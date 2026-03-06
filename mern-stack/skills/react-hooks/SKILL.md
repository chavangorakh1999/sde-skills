---
name: react-hooks
description: "Hook patterns: useEffect dependency array correctness, useMemo/useCallback tradeoffs, custom hook extraction, avoiding stale closures, common pitfalls. Use when writing or debugging React hooks."
---

## React Hooks

Hooks are powerful but have subtle rules. Most React bugs come from: wrong useEffect dependencies, stale closures, and premature optimization with useMemo/useCallback.

### Context

Hook problem or pattern to address: **$ARGUMENTS**

---

### useEffect — The Most Misused Hook

```jsx
// Rule: every value from the component scope that useEffect reads must be in the dependency array
// React's exhaustive-deps ESLint rule catches most violations — enable it

// BAD: missing dependency
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
    // userId is read but not in deps — stale closure when userId changes
  }, []);  // runs only once — BUG: doesn't refetch when userId changes

  // GOOD: include userId in deps
  useEffect(() => {
    let isMounted = true;
    fetchUser(userId).then(user => {
      if (isMounted) setUser(user);  // prevent state update on unmounted component
    });
    return () => { isMounted = false; };  // cleanup
  }, [userId]);  // re-runs when userId changes
}

// Pattern: async function inside useEffect (can't make the callback async)
useEffect(() => {
  // Can't: useEffect(() => async () => {}, [])
  // Do: define async function inside and call it

  async function loadUser() {
    try {
      setLoading(true);
      const data = await fetchUser(userId);
      setUser(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }

  loadUser();
}, [userId]);

// Cleanup: subscriptions, timers, event listeners
useEffect(() => {
  const subscription = websocket.subscribe(channel, handleMessage);

  return () => {
    subscription.unsubscribe();  // cleanup on unmount or channel change
  };
}, [channel, handleMessage]);  // handleMessage must be stable (useCallback) to avoid re-subscribing
```

---

### Stale Closures

```jsx
// Stale closure: a function "closes over" a variable that changes later,
// but the function still sees the old value

// BAD: stale counter in setInterval
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count);  // always logs 0 — stale closure!
      setCount(count + 1); // always sets to 1 — stale closure!
    }, 1000);
    return () => clearInterval(interval);
  }, []);  // empty deps: interval never sees updated count

  // FIX 1: Use functional update (doesn't need current state from closure)
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(prev => prev + 1);  // callback form: always gets latest state
    }, 1000);
    return () => clearInterval(interval);
  }, []);

  // FIX 2: Add count to deps (re-creates interval on every change — often wasteful)
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(interval);
  }, [count]);

  // FIX 3: useRef for values that shouldn't trigger re-renders
  const countRef = useRef(count);
  countRef.current = count;  // always current, but doesn't trigger re-render
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(countRef.current + 1);
    }, 1000);
    return () => clearInterval(interval);
  }, []);
}
```

---

### useMemo and useCallback — Don't Overuse

```jsx
// useMemo: memoize an expensive computation
// useCallback: memoize a function reference

// ONLY use useMemo when:
// 1. The computation is expensive (> 1ms — measure with console.time)
// 2. The result is used in a dependency array

// ONLY use useCallback when:
// 1. The function is in a dependency array of useEffect, useMemo, useCallback
// 2. The function is passed to a React.memo-wrapped child

// Premature optimization: wrapping everything in useMemo/useCallback
// React re-renders are usually fast. Measure before optimizing.

// BAD: useMemo on cheap computation
const doubled = useMemo(() => count * 2, [count]);  // just write: const doubled = count * 2;

// GOOD: useMemo on genuinely expensive computation
const expensiveResult = useMemo(
  () => largeArray.filter(item => heavyComputation(item, filter)),
  [largeArray, filter]  // only recomputes when these change
);

// GOOD: useCallback for stable function reference in dependency array
const handleSearch = useCallback(
  async (query) => {
    const results = await searchApi(query);
    setResults(results);
  },
  []  // no deps: function never changes
);

useEffect(() => {
  handleSearch(debouncedQuery);  // handleSearch in deps — must be stable
}, [debouncedQuery, handleSearch]);
```

---

### Custom Hooks — Extraction Pattern

```jsx
// Extract when: the same stateful logic appears in 2+ components
// Custom hooks = reusable stateful logic (not reusable UI)

// Before: logic repeated in multiple components
function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetchUser(userId)
      .then(setUser)
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, [userId]);

  // same logic in UserCard, UserList, UserSettings...
}

// After: custom hook
function useUser(userId) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);

    fetchUser(userId)
      .then(data => { if (!cancelled) setUser(data); })
      .catch(err => { if (!cancelled) setError(err.message); })
      .finally(() => { if (!cancelled) setLoading(false); });

    return () => { cancelled = true; };
  }, [userId]);

  return { user, loading, error };
}

// Usage:
function UserProfile({ userId }) {
  const { user, loading, error } = useUser(userId);
  // Clean component — just UI logic
}

// More custom hook patterns:
// useLocalStorage — persists state to localStorage
// useDebounce — debounce a rapidly-changing value
// usePrevious — access previous render's value
// useIntersectionObserver — lazy loading / infinite scroll trigger
// useWindowSize — responsive behavior
```

---

### Common Hook Anti-Patterns

```jsx
// 1. Calling hooks conditionally
// BAD
function Component({ isLoggedIn }) {
  if (isLoggedIn) {
    const [data] = useData();  // WRONG: hooks can't be in conditionals
  }
}

// GOOD: hook always called, condition inside the hook
function Component({ isLoggedIn }) {
  const [data] = useData({ skip: !isLoggedIn });
}

// 2. Creating objects/arrays in render that cause dependency thrashing
function Component({ userId }) {
  const options = { userId, page: 1 };  // new object every render!

  useEffect(() => {
    fetchData(options);  // options is "new" every render
  }, [options]);  // runs on EVERY render

  // Fix: depend on the primitive values
  useEffect(() => {
    fetchData({ userId, page: 1 });
  }, [userId]);  // only runs when userId changes

// 3. useEffect for synchronous state derivation
// BAD: useEffect to compute derived state
const [doubled, setDoubled] = useState(0);
useEffect(() => {
  setDoubled(count * 2);  // unnecessary intermediate render
}, [count]);

// GOOD: compute inline (no effect needed)
const doubled = count * 2;  // computed during render, no extra render

// 4. Missing cleanup for subscriptions (memory leak)
useEffect(() => {
  const unsubscribe = store.subscribe(handleChange);
  // MISSING: return () => unsubscribe(); <- memory leak!
}, []);

useEffect(() => {
  const unsubscribe = store.subscribe(handleChange);
  return () => unsubscribe();  // always cleanup subscriptions
}, [handleChange]);
```

---

### Output Format

```
## Hook Analysis: [Component/Problem]

### Issues Found
[For each: which hook, what's wrong, why it causes problems]

### Correct Implementation
[Before/after code for each fix]

### Custom Hook Extraction (if applicable)
[When logic should be extracted to a custom hook]
```
