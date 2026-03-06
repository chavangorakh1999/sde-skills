---
name: jwt-auth
description: "Complete JWT auth flow: registration, login, access + refresh token pair, httpOnly cookies, token rotation on refresh, logout (server-side invalidation), silent refresh. Use when implementing auth in a MERN app."
---

## JWT Authentication (MERN)

Production JWT auth uses short-lived access tokens (15 min) paired with long-lived refresh tokens (30 days). Access tokens are in memory (frontend) or Authorization headers. Refresh tokens in httpOnly cookies.

### Context

Auth requirements: **$ARGUMENTS**

---

### The Full Auth Flow

```
Register:
  POST /auth/register {email, password, displayName}
  -> hash password (bcrypt)
  -> create user
  -> return access token + set refresh token cookie

Login:
  POST /auth/login {email, password}
  -> find user by email
  -> verify password (bcrypt.compare)
  -> generate access token (15 min) + refresh token (30 days)
  -> store hashed refresh token in DB
  -> return access token in body + refresh token in httpOnly cookie

Authenticated request:
  GET /api/v1/users/me
  Authorization: Bearer <access_token>
  -> verify JWT signature + expiry + audience
  -> attach req.user = decoded payload
  -> proceed

Refresh:
  POST /auth/refresh
  Cookie: refreshToken=<token>
  -> verify refresh token cookie
  -> find hashed token in DB (confirm not revoked)
  -> rotate: generate new refresh token, invalidate old one
  -> return new access token + new refresh token cookie

Logout:
  POST /auth/logout
  Cookie: refreshToken=<token>
  -> delete refresh token from DB
  -> clear refresh token cookie
```

---

### Backend Implementation

```javascript
// utils/jwt.js
import jwt from 'jsonwebtoken';
import { config } from '../config/index.js';

export function generateAccessToken(userId, role) {
  return jwt.sign(
    { sub: userId, role },
    config.jwt.accessSecret,
    {
      expiresIn: config.jwt.accessExpiresIn,  // '15m'
      audience: 'sde-skills-api',
      issuer: 'sde-skills-auth'
    }
  );
}

export function generateRefreshToken() {
  // Random opaque token (not JWT — harder to decode and misuse)
  return crypto.randomBytes(32).toString('hex');
}

export function verifyAccessToken(token) {
  return jwt.verify(token, config.jwt.accessSecret, {
    audience: 'sde-skills-api',
    issuer: 'sde-skills-auth'
  });
}

// services/auth.service.js
export class AuthService {
  constructor(userRepo, sessionRepo) {
    this.userRepo = userRepo;
    this.sessionRepo = sessionRepo;
  }

  async register({ email, password, displayName }) {
    const existing = await this.userRepo.findByEmail(email);
    if (existing) throw new ConflictError('Email already registered');

    const passwordHash = await bcrypt.hash(password, 12);
    const user = await this.userRepo.create({ email, passwordHash, displayName });

    return this.#issueTokens(user);
  }

  async login({ email, password }) {
    const user = await this.userRepo.findByEmailWithPassword(email);
    if (!user) throw new UnauthorizedError('Invalid credentials');

    const passwordMatch = await bcrypt.compare(password, user.passwordHash);
    if (!passwordMatch) throw new UnauthorizedError('Invalid credentials');

    if (!user.emailVerified) throw new ForbiddenError('Please verify your email');

    return this.#issueTokens(user);
  }

  async refresh(rawRefreshToken) {
    // Find session by hashed token
    const tokenHash = this.#hashToken(rawRefreshToken);
    const session = await this.sessionRepo.findByToken(tokenHash);

    if (!session || session.expiresAt < new Date()) {
      // Possible reuse attack: if token was used and then reused, revoke all sessions
      if (session?.userId) await this.sessionRepo.deleteAllForUser(session.userId);
      throw new UnauthorizedError('Invalid or expired refresh token');
    }

    // Rotate: delete old, create new
    await this.sessionRepo.delete(session._id);
    const user = await this.userRepo.findById(session.userId);
    return this.#issueTokens(user);
  }

  async logout(rawRefreshToken) {
    const tokenHash = this.#hashToken(rawRefreshToken);
    await this.sessionRepo.deleteByToken(tokenHash);
  }

  #issueTokens(user) {
    const accessToken = generateAccessToken(user._id.toString(), user.role);
    const refreshToken = generateRefreshToken();
    const tokenHash = this.#hashToken(refreshToken);

    // Store hashed refresh token in DB
    this.sessionRepo.create({
      userId: user._id,
      tokenHash,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)  // 30 days
    });

    return { accessToken, refreshToken, user: sanitizeUser(user) };
  }

  #hashToken(token) {
    return crypto.createHash('sha256').update(token).digest('hex');
  }
}

// controllers/auth.controller.js
const REFRESH_COOKIE_OPTIONS = {
  httpOnly: true,               // not accessible by JS
  secure: process.env.NODE_ENV === 'production',  // HTTPS only in prod
  sameSite: 'strict',           // CSRF protection
  maxAge: 30 * 24 * 60 * 60 * 1000,  // 30 days in ms
  path: '/api/v1/auth'          // only sent on auth routes
};

export class AuthController {
  constructor(authService) { this.authService = authService; }

  register = async (req, res, next) => {
    try {
      const { accessToken, refreshToken, user } = await this.authService.register(req.body);
      res.cookie('refreshToken', refreshToken, REFRESH_COOKIE_OPTIONS);
      res.status(201).json({ data: { accessToken, user } });
    } catch (err) { next(err); }
  };

  login = async (req, res, next) => {
    try {
      const { accessToken, refreshToken, user } = await this.authService.login(req.body);
      res.cookie('refreshToken', refreshToken, REFRESH_COOKIE_OPTIONS);
      res.json({ data: { accessToken, user } });
    } catch (err) { next(err); }
  };

  refresh = async (req, res, next) => {
    try {
      const rawToken = req.cookies.refreshToken;
      if (!rawToken) throw new UnauthorizedError('Refresh token required');
      const { accessToken, refreshToken } = await this.authService.refresh(rawToken);
      res.cookie('refreshToken', refreshToken, REFRESH_COOKIE_OPTIONS);
      res.json({ data: { accessToken } });
    } catch (err) { next(err); }
  };

  logout = async (req, res, next) => {
    try {
      const rawToken = req.cookies.refreshToken;
      if (rawToken) await this.authService.logout(rawToken);
      res.clearCookie('refreshToken', { ...REFRESH_COOKIE_OPTIONS, maxAge: 0 });
      res.status(204).end();
    } catch (err) { next(err); }
  };
}

// middlewares/authenticate.js
export async function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: { code: 'MISSING_TOKEN' } });

  try {
    const payload = verifyAccessToken(token);
    req.user = { id: payload.sub, role: payload.role };
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: { code: 'TOKEN_EXPIRED' } });
    }
    return res.status(401).json({ error: { code: 'INVALID_TOKEN' } });
  }
}
```

---

### Frontend (React + Axios)

```javascript
// api/apiClient.js — axios instance with interceptors
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL ?? 'http://localhost:3000/api/v1',
  withCredentials: true  // send cookies (for refresh token)
});

// Request interceptor: attach access token from memory
apiClient.interceptors.request.use((config) => {
  const token = authStore.getState().accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor: handle 401 (token expired) -> refresh
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => error ? prom.reject(error) : prom.resolve(token));
  failedQueue = [];
};

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue the request while refresh is in progress
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return apiClient(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const { data } = await axios.post('/api/v1/auth/refresh', {}, { withCredentials: true });
        const newToken = data.data.accessToken;
        authStore.getState().setAccessToken(newToken);
        processQueue(null, newToken);
        originalRequest.headers.Authorization = `Bearer ${newToken}`;
        return apiClient(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        authStore.getState().clearAuth();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);

export default apiClient;
```

---

### Output Format

```
## JWT Auth Implementation: [App Name]

### Flow Diagram
[Registration -> Login -> Refresh -> Logout]

### Backend Components
[auth.service.js, auth.controller.js, authenticate.js middleware]

### Frontend Components
[apiClient.js with interceptors, useAuth hook]

### Session Model
[Mongoose schema for storing hashed refresh tokens]

### Security Checklist
[ ] Passwords hashed with bcrypt (work factor 12)
[ ] Refresh tokens stored hashed (SHA-256)
[ ] Refresh token rotation on every refresh
[ ] Reuse detection: revoke all sessions on reuse attempt
[ ] httpOnly cookie for refresh token
[ ] Short access token expiry (15 min)
[ ] audience + issuer validated on JWT verification
```
