---
name: secret-management
description: "Secrets management: AWS Secrets Manager, HashiCorp Vault, environment variable patterns, secret rotation, Kubernetes secrets, and detecting leaked secrets. Use when designing or auditing secret handling."
---

## Secret Management

### Context

Secret management problem or system: **$ARGUMENTS**

---

### The Rules

```
1. NEVER commit secrets to git (not even encrypted ones in the repo)
2. NEVER log secret values
3. NEVER pass secrets via CLI args (visible in process list: ps aux)
4. NEVER store secrets in plain environment variables in Dockerfiles
5. Rotate secrets when a team member leaves or a breach is suspected
6. Principle of least privilege: each service only gets secrets it needs
```

---

### Local Development (.env Pattern)

```bash
# .env.example — COMMIT THIS (template with placeholder values)
NODE_ENV=development
PORT=3000
MONGODB_URI=mongodb://localhost:27017/myapp
REDIS_URL=redis://localhost:6379
JWT_ACCESS_SECRET=<generate: openssl rand -hex 32>
JWT_REFRESH_SECRET=<generate: openssl rand -hex 32>
SENDGRID_API_KEY=<get from SendGrid dashboard>

# .env — DO NOT COMMIT (add to .gitignore)
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/myapp
JWT_ACCESS_SECRET=a1b2c3d4e5f6...  # actual value here

# .gitignore
.env
.env.local
.env.*.local
*.pem
*.key
```

```javascript
// Validate all required env vars at startup — fail fast
import Joi from 'joi';

const schema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'test', 'production').required(),
  MONGODB_URI: Joi.string().uri().required(),
  JWT_ACCESS_SECRET: Joi.string().min(32).required(),
  JWT_REFRESH_SECRET: Joi.string().min(32).required(),
  REDIS_URL: Joi.string().uri().required(),
}).unknown(true);

const { error, value: env } = schema.validate(process.env);
if (error) {
  console.error('FATAL: Invalid environment config:', error.message);
  process.exit(1);
}
```

---

### AWS Secrets Manager

```javascript
// src/config/secrets.js — load secrets from AWS Secrets Manager at startup
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: process.env.AWS_REGION });

async function getSecret(secretId) {
  const response = await client.send(
    new GetSecretValueCommand({ SecretId: secretId })
  );

  if (response.SecretString) {
    return JSON.parse(response.SecretString);
  }
  return JSON.parse(Buffer.from(response.SecretBinary, 'base64').toString());
}

// Load all secrets at startup (not per-request — avoid latency and rate limits)
let secrets = null;

export async function loadSecrets() {
  if (secrets) return secrets;  // cached after first load

  const appSecrets = await getSecret(`${process.env.APP_NAME}/${process.env.NODE_ENV}`);

  secrets = {
    jwtAccessSecret: appSecrets.JWT_ACCESS_SECRET,
    jwtRefreshSecret: appSecrets.JWT_REFRESH_SECRET,
    mongodbUri: appSecrets.MONGODB_URI,
    sendgridApiKey: appSecrets.SENDGRID_API_KEY,
  };

  return secrets;
}

// server.js — load before starting
import { loadSecrets } from './config/secrets.js';

const secrets = await loadSecrets();
// Now pass to services rather than reading from process.env
```

---

### AWS SSM Parameter Store

```bash
# Store secrets:
aws ssm put-parameter \
  --name "/myapp/production/JWT_ACCESS_SECRET" \
  --value "$(openssl rand -hex 32)" \
  --type SecureString \
  --key-id alias/aws/ssm  # KMS key for encryption

# Read in application (Node.js):
import { SSMClient, GetParameterCommand } from '@aws-sdk/client-ssm';

const ssm = new SSMClient({ region: process.env.AWS_REGION });

async function getParameter(name) {
  const response = await ssm.send(new GetParameterCommand({
    Name: name,
    WithDecryption: true,   // decrypt SecureString automatically
  }));
  return response.Parameter.Value;
}

# Batch fetch (reduces API calls):
import { GetParametersCommand } from '@aws-sdk/client-ssm';

async function getParameters(names) {
  const response = await ssm.send(new GetParametersCommand({
    Names: names,
    WithDecryption: true,
  }));
  return Object.fromEntries(
    response.Parameters.map(p => [p.Name.split('/').pop(), p.Value])
  );
}
```

---

### Kubernetes Secrets

```yaml
# Create secret imperatively (don't put in git):
# kubectl create secret generic api-secrets \
#   --from-literal=JWT_ACCESS_SECRET=$(openssl rand -hex 32) \
#   --from-literal=MONGODB_URI=mongodb+srv://...

# Use External Secrets Operator to sync from AWS/GCP/Vault:
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: api-secrets            # creates this K8s Secret
    creationPolicy: Owner
  data:
    - secretKey: JWT_ACCESS_SECRET
      remoteRef:
        key: myapp/production
        property: JWT_ACCESS_SECRET
    - secretKey: MONGODB_URI
      remoteRef:
        key: myapp/production
        property: MONGODB_URI
```

---

### Secret Rotation

```javascript
// Rotation strategy for JWT secrets (zero-downtime):
// 1. Add JWT_ACCESS_SECRET_OLD = old value
// 2. Deploy: verify with new secret, fall back to old on failure
// 3. Wait until all old tokens expire (access token TTL = 15min)
// 4. Remove JWT_ACCESS_SECRET_OLD

// jwt.js with rotation support
function verifyAccessToken(token) {
  // Try new secret first
  try {
    return jwt.verify(token, config.jwt.accessSecret);
  } catch {
    // Fall back to old secret during rotation window
    if (config.jwt.accessSecretOld) {
      return jwt.verify(token, config.jwt.accessSecretOld);
    }
    throw err;
  }
}

// AWS Secrets Manager has built-in rotation:
aws secretsmanager rotate-secret --secret-id myapp/production \
  --rotation-lambda-arn arn:aws:lambda:...:function:RotateSecret
```

---

### Detecting Leaked Secrets

```bash
# Pre-commit: detect before commit
brew install git-secrets
git secrets --install  # installs pre-commit hook
git secrets --register-aws  # add AWS patterns

# Or use gitleaks:
brew install gitleaks
gitleaks detect --source . --verbose

# Add to CI:
- name: Check for leaked secrets
  uses: gitleaks/gitleaks-action@v2

# Scan git history (if you suspect a past commit):
gitleaks detect --source . --log-opts="--all" --verbose

# If found: rotate the secret immediately, then remove from history
git filter-repo --path .env --invert-paths  # remove file from history
# Force push (requires all team members to re-clone)
```

---

### Secret Management Checklist

```
Development:
  - [ ] .env in .gitignore
  - [ ] .env.example committed with placeholders
  - [ ] Secrets validated at startup (fail fast)

CI/CD:
  - [ ] Secrets stored in GitHub Actions Secrets (not in workflow files)
  - [ ] Gitleaks or git-secrets runs on every PR
  - [ ] Secrets masked in CI logs

Production:
  - [ ] Secrets stored in AWS Secrets Manager / Vault / SSM
  - [ ] IAM roles scoped to only the secrets each service needs
  - [ ] Secret rotation schedule defined (90 days max)
  - [ ] Access logs enabled on secret store
  - [ ] Alerting on unexpected secret access patterns
```
