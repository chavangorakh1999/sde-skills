---
name: react-components
description: "React component patterns: composition, compound components, render props, controlled vs uncontrolled, accessibility, error boundaries, and performance optimization. Use when designing or refactoring React components."
---

## React Component Patterns

### Context

Component design problem or pattern: **$ARGUMENTS**

---

### Component Composition Over Inheritance

```jsx
// Anti-pattern: prop drilling through many layers
function Page({ user, onLogout, theme, notifications }) {
  return <Header user={user} onLogout={onLogout} theme={theme} notifications={notifications} />;
}

// Pattern: composition with children
function Card({ children, className = '' }) {
  return <div className={`card ${className}`}>{children}</div>;
}

function CardHeader({ children }) {
  return <div className="card-header">{children}</div>;
}

function CardBody({ children }) {
  return <div className="card-body">{children}</div>;
}

// Usage — flexible, no prop drilling
function UserCard({ user }) {
  return (
    <Card>
      <CardHeader>
        <Avatar src={user.avatar} size="md" />
        <h2>{user.displayName}</h2>
      </CardHeader>
      <CardBody>
        <UserStats userId={user.id} />
      </CardBody>
    </Card>
  );
}
```

---

### Compound Components

```jsx
// Components that share implicit state — like <select> + <option>
// tabs/Tabs.jsx
import { createContext, useContext, useState } from 'react';

const TabsContext = createContext(null);

function Tabs({ children, defaultTab }) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div role="tablist" className="tab-list">{children}</div>;
}

function Tab({ id, children }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  const isActive = activeTab === id;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      aria-controls={`panel-${id}`}
      className={`tab ${isActive ? 'tab--active' : ''}`}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }) {
  const { activeTab } = useContext(TabsContext);
  if (activeTab !== id) return null;

  return (
    <div role="tabpanel" id={`panel-${id}`} className="tab-panel">
      {children}
    </div>
  );
}

// Attach as static properties for clean API
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage:
function ProfilePage() {
  return (
    <Tabs defaultTab="overview">
      <Tabs.List>
        <Tabs.Tab id="overview">Overview</Tabs.Tab>
        <Tabs.Tab id="settings">Settings</Tabs.Tab>
        <Tabs.Tab id="activity">Activity</Tabs.Tab>
      </Tabs.List>
      <Tabs.Panel id="overview"><OverviewContent /></Tabs.Panel>
      <Tabs.Panel id="settings"><SettingsContent /></Tabs.Panel>
      <Tabs.Panel id="activity"><ActivityContent /></Tabs.Panel>
    </Tabs>
  );
}
```

---

### Controlled vs Uncontrolled

```jsx
// Controlled — React owns the value (recommended for most cases)
function SearchInput({ value, onChange, placeholder }) {
  return (
    <input
      type="search"
      value={value}          // controlled by parent
      onChange={e => onChange(e.target.value)}
      placeholder={placeholder}
    />
  );
}

// Uncontrolled — DOM owns the value (use for file inputs, avoid otherwise)
import { useRef } from 'react';

function FileUpload({ onUpload }) {
  const fileInputRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    const file = fileInputRef.current.files[0];
    if (file) onUpload(file);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={fileInputRef} type="file" accept="image/*" />
      <button type="submit">Upload</button>
    </form>
  );
}

// Dual-mode (optional controlled): flexible component
function TextInput({ value, defaultValue, onChange, ...props }) {
  const isControlled = value !== undefined;
  const [localValue, setLocalValue] = useState(defaultValue ?? '');

  const currentValue = isControlled ? value : localValue;

  const handleChange = (e) => {
    if (!isControlled) setLocalValue(e.target.value);
    onChange?.(e.target.value);
  };

  return <input value={currentValue} onChange={handleChange} {...props} />;
}
```

---

### Error Boundaries

```jsx
// Error boundaries must be class components (no hook equivalent yet)
import { Component } from 'react';

class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    // Log to error tracking (Sentry, etc.)
    console.error('ErrorBoundary caught:', error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div role="alert">
          <h2>Something went wrong.</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage — wrap at appropriate granularity
function Dashboard() {
  return (
    <div>
      <ErrorBoundary fallback={<p>Failed to load activity</p>}>
        <ActivityFeed />         {/* isolated — error here won't crash whole page */}
      </ErrorBoundary>
      <ErrorBoundary fallback={<p>Failed to load stats</p>}>
        <StatsWidget />
      </ErrorBoundary>
    </div>
  );
}
```

---

### List Performance

```jsx
// Keys must be stable, unique, and not index-based (when list can reorder)
// Anti-pattern:
{users.map((user, i) => <UserCard key={i} user={user} />)}  // key=index breaks reconciliation

// Correct:
{users.map(user => <UserCard key={user.id} user={user} />)}

// Virtualize long lists with react-window
import { FixedSizeList } from 'react-window';

function UserList({ users }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      <UserCard user={users[index]} />
    </div>
  );

  return (
    <FixedSizeList
      height={600}
      itemCount={users.length}
      itemSize={80}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

---

### Accessibility Essentials

```jsx
// Modal with focus trap and ARIA
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, title, children }) {
  const modalRef = useRef(null);

  useEffect(() => {
    if (!isOpen) return;

    // Focus first focusable element on open
    const focusable = modalRef.current?.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    focusable?.[0]?.focus();

    // Close on Escape
    const handleKey = (e) => {
      if (e.key === 'Escape') onClose();
    };

    document.addEventListener('keydown', handleKey);
    return () => document.removeEventListener('keydown', handleKey);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose} aria-hidden="true">
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        className="modal"
        onClick={e => e.stopPropagation()}  // prevent overlay click from closing when clicking modal
      >
        <h2 id="modal-title">{title}</h2>
        {children}
        <button onClick={onClose} aria-label="Close dialog">x</button>
      </div>
    </div>,
    document.body
  );
}
```

---

### Component Checklist

```
## Component Review: [ComponentName]

### Structure
- [ ] Single responsibility — does one thing
- [ ] Props interface is minimal and explicit
- [ ] No prop drilling beyond 2 levels — use context or composition

### Performance
- [ ] Keys are stable IDs (not array index)
- [ ] Expensive components wrapped in React.memo where appropriate
- [ ] Lists > 100 items use virtualization

### Accessibility
- [ ] Interactive elements are keyboard accessible
- [ ] Images have alt text
- [ ] Forms have associated labels
- [ ] Dynamic content announced via aria-live or role="alert"

### Error Handling
- [ ] Error boundary wraps async-dependent sections
- [ ] Loading and error states rendered
```
