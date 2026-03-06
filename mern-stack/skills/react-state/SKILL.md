---
name: react-state
description: "React state management: useState, useReducer, Context API, Zustand, React Query for server state. Covers when to use each, anti-patterns, and optimistic updates. Use when designing state architecture for React apps."
---

## React State Management

### Context

State management problem or question: **$ARGUMENTS**

---

### State Colocation Principle

```
Rule: Keep state as close to where it's used as possible.

Local UI state  → useState in the component
Shared UI state → lift to nearest common ancestor, or Zustand
Server state    → React Query (TanStack Query)
URL state       → search params (useSearchParams)
Form state      → React Hook Form

Do NOT put server data in global state. React Query is the server state layer.
```

---

### useState vs useReducer

```jsx
// useState — simple values, independent state
const [count, setCount] = useState(0);
const [isOpen, setIsOpen] = useState(false);

// useReducer — when next state depends on previous, or multiple related fields
// Anti-pattern with useState:
const [loading, setLoading] = useState(false);
const [data, setData] = useState(null);
const [error, setError] = useState(null);
// These three can get out of sync — "loading: false, data: null, error: null" is ambiguous

// Pattern: useReducer for state machine
const initialState = { status: 'idle', data: null, error: null };

function reducer(state, action) {
  switch (action.type) {
    case 'FETCH_START':
      return { status: 'loading', data: null, error: null };
    case 'FETCH_SUCCESS':
      return { status: 'success', data: action.payload, error: null };
    case 'FETCH_ERROR':
      return { status: 'error', data: null, error: action.payload };
    case 'RESET':
      return initialState;
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function useAsyncData(fetchFn) {
  const [state, dispatch] = useReducer(reducer, initialState);

  const execute = useCallback(async () => {
    dispatch({ type: 'FETCH_START' });
    try {
      const data = await fetchFn();
      dispatch({ type: 'FETCH_SUCCESS', payload: data });
    } catch (err) {
      dispatch({ type: 'FETCH_ERROR', payload: err.message });
    }
  }, [fetchFn]);

  return { ...state, execute };
}
```

---

### Context API — Shared UI State

```jsx
// For theme, auth user, language — NOT server data
// auth/AuthContext.jsx
import { createContext, useContext, useReducer, useEffect } from 'react';

const AuthContext = createContext(null);

function authReducer(state, action) {
  switch (action.type) {
    case 'LOGIN':    return { user: action.payload, status: 'authenticated' };
    case 'LOGOUT':   return { user: null, status: 'unauthenticated' };
    case 'LOADING':  return { ...state, status: 'loading' };
    default:         return state;
  }
}

export function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(authReducer, {
    user: null,
    status: 'loading'  // loading until we verify token
  });

  useEffect(() => {
    // Restore session on mount
    authService.getMe()
      .then(user => dispatch({ type: 'LOGIN', payload: user }))
      .catch(() => dispatch({ type: 'LOGOUT' }));
  }, []);

  const login = useCallback(async (credentials) => {
    dispatch({ type: 'LOADING' });
    const user = await authService.login(credentials);
    dispatch({ type: 'LOGIN', payload: user });
  }, []);

  const logout = useCallback(async () => {
    await authService.logout();
    dispatch({ type: 'LOGOUT' });
  }, []);

  // Memoize value object to prevent unnecessary re-renders
  const value = useMemo(() => ({
    user: state.user,
    status: state.status,
    isAuthenticated: state.status === 'authenticated',
    login,
    logout,
  }), [state, login, logout]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}

// Performance: split context if some consumers only need part of state
// AuthUserContext (user object) + AuthActionsContext (login/logout)
// Consumers only re-render when their specific context changes
```

---

### Zustand — Lightweight Global State

```jsx
// For client-side global state that doesn't fit local or Context
// store/cartStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

const useCartStore = create(
  persist(
    (set, get) => ({
      items: [],

      addItem: (product, quantity = 1) => {
        set(state => {
          const existing = state.items.find(i => i.productId === product.id);
          if (existing) {
            return {
              items: state.items.map(i =>
                i.productId === product.id
                  ? { ...i, quantity: i.quantity + quantity }
                  : i
              )
            };
          }
          return {
            items: [...state.items, { productId: product.id, name: product.name, price: product.price, quantity }]
          };
        });
      },

      removeItem: (productId) => {
        set(state => ({ items: state.items.filter(i => i.productId !== productId) }));
      },

      clearCart: () => set({ items: [] }),

      // Derived state as selector
      get total() {
        return get().items.reduce((sum, item) => sum + item.price * item.quantity, 0);
      },

      get itemCount() {
        return get().items.reduce((sum, item) => sum + item.quantity, 0);
      }
    }),
    {
      name: 'cart-storage',    // localStorage key
      partialize: (state) => ({ items: state.items }),  // only persist items
    }
  )
);

// Usage — select only what you need to avoid unnecessary re-renders
function CartIcon() {
  const itemCount = useCartStore(state => state.itemCount);
  return <span>{itemCount}</span>;
}

function CartPage() {
  const { items, removeItem, total } = useCartStore(state => ({
    items: state.items,
    removeItem: state.removeItem,
    total: state.total,
  }));
  // ...
}
```

---

### React Query — Server State

```jsx
// Install: npm i @tanstack/react-query
// queryClient.js
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,   // 5 min — don't refetch if fresh
      gcTime: 10 * 60 * 1000,     // 10 min — garbage collect
      retry: 2,
      refetchOnWindowFocus: true,
    },
  },
});

// hooks/useUsers.js — typed, cached, auto-refetch
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { userApi } from '../api/userApi.js';

export function useUsers(params) {
  return useQuery({
    queryKey: ['users', params],     // params in key — cache per param set
    queryFn: () => userApi.list(params),
  });
}

export function useUser(id) {
  return useQuery({
    queryKey: ['users', id],
    queryFn: () => userApi.get(id),
    enabled: Boolean(id),            // don't fetch if no id
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }) => userApi.update(id, data),

    // Optimistic update
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: ['users', id] });
      const previous = queryClient.getQueryData(['users', id]);

      queryClient.setQueryData(['users', id], old => ({ ...old, ...data }));

      return { previous };  // context for onError rollback
    },

    onError: (err, { id }, context) => {
      queryClient.setQueryData(['users', id], context.previous);
    },

    onSettled: (data, err, { id }) => {
      // Always refetch after mutation to sync with server
      queryClient.invalidateQueries({ queryKey: ['users', id] });
      queryClient.invalidateQueries({ queryKey: ['users'] });
    }
  });
}

// Component usage:
function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useUser(userId);
  const { mutate: updateUser, isPending } = useUpdateUser();

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <form onSubmit={e => {
      e.preventDefault();
      updateUser({ id: userId, data: { displayName: e.target.displayName.value } });
    }}>
      <input name="displayName" defaultValue={user.displayName} />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

---

### Form State — React Hook Form

```jsx
// npm i react-hook-form @hookform/resolvers zod
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

function LoginForm({ onSubmit }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setError,
  } = useForm({ resolver: zodResolver(schema) });

  const onSubmitHandler = async (data) => {
    try {
      await onSubmit(data);
    } catch (err) {
      // Map server errors to fields
      if (err.code === 'INVALID_CREDENTIALS') {
        setError('password', { message: 'Invalid email or password' });
      }
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmitHandler)}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} />
        {errors.email && <span role="alert">{errors.email.message}</span>}
      </div>
      <div>
        <label htmlFor="password">Password</label>
        <input id="password" type="password" {...register('password')} />
        {errors.password && <span role="alert">{errors.password.message}</span>}
      </div>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Log in'}
      </button>
    </form>
  );
}
```

---

### State Decision Tree

```
Is it server data? (from an API)
  → Yes: React Query

Is it form data?
  → Yes: React Hook Form

Is it URL-representable? (filters, pagination, current tab)
  → Yes: useSearchParams / URL state

Is it used in only one component or its direct children?
  → Yes: useState / useReducer locally

Is it shared UI state across distant components?
  → Few consumers: Context API
  → Complex or frequent updates: Zustand
```
