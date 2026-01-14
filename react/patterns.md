# React Patterns – Advanced Component Design

> **Mục tiêu:** Hiểu sâu các React patterns từ legacy đến modern, biết khi nào dùng pattern nào, và cách implement production-ready.

---

## 📚 Overview - Pattern Landscape

```
React Patterns Timeline:

2013-2016: Class Components Era
├── Higher-Order Components (HOC)
└── Render Props

2018+: Hooks Era
├── Hooks Composition ⭐ Modern
├── Compound Components ⭐ Modern
└── Controlled/Uncontrolled ⭐ Fundamental

Legacy patterns: Biết để đọc code cũ, đừng dùng cho code mới
Modern patterns: Dùng cho projects mới
```

---

## 1. Compound Components Pattern

### Concept

> **Compound Components = Components hoạt động cùng nhau để tạo nên một functionality hoàn chỉnh.**

**Mental model:** HTML `<select>` + `<option>`

```html
<!-- HTML native compound component -->
<select>
  <option value="1">Option 1</option>
  <option value="2">Option 2</option>
</select>
```

React compound components works tương tự: Parent component share state với children qua Context.

---

### Basic Implementation

```jsx
import { createContext, useContext, useState } from 'react';

// 1. Create context
const TabContext = createContext();

// 2. Parent component (manages state)
function Tabs({ children, defaultValue }) {
  const [activeTab, setActiveTab] = useState(defaultValue);
  
  return (
    <TabContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabContext.Provider>
  );
}

// 3. Child components (consume context)
function TabList({ children }) {
  return <div className="tab-list">{children}</div>;
}

function Tab({ value, children }) {
  const { activeTab, setActiveTab } = useContext(TabContext);
  const isActive = activeTab === value;
  
  return (
    <button
      className={`tab ${isActive ? 'active' : ''}`}
      onClick={() => setActiveTab(value)}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }) {
  const { activeTab } = useContext(TabContext);
  
  if (activeTab !== value) return null;
  
  return <div className="tab-panel">{children}</div>;
}

// 4. Attach sub-components to parent
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

export default Tabs;
```

---

### Usage

```jsx
function App() {
  return (
    <Tabs defaultValue="profile">
      <Tabs.List>
        <Tabs.Tab value="profile">Profile</Tabs.Tab>
        <Tabs.Tab value="settings">Settings</Tabs.Tab>
        <Tabs.Tab value="notifications">Notifications</Tabs.Tab>
      </Tabs.List>
      
      <Tabs.Panel value="profile">
        <ProfileContent />
      </Tabs.Panel>
      
      <Tabs.Panel value="settings">
        <SettingsContent />
      </Tabs.Panel>
      
      <Tabs.Panel value="notifications">
        <NotificationsContent />
      </Tabs.Panel>
    </Tabs>
  );
}
```

---

### Pros & Cons

✅ **Pros:**
- Flexible composition (user controls structure)
- Clean API (readable như HTML)
- Encapsulation (state hidden inside)
- Extensible (easy to add new variants)

❌ **Cons:**
- Implicit coupling (components phải là children)
- Context overhead (re-renders)
- Learning curve (phải hiểu pattern)

---

### Real Example: Accordion

```jsx
const AccordionContext = createContext();

function Accordion({ children, allowMultiple = false }) {
  const [openItems, setOpenItems] = useState([]);
  
  const toggleItem = (id) => {
    setOpenItems(prev => {
      if (allowMultiple) {
        return prev.includes(id)
          ? prev.filter(item => item !== id)
          : [...prev, id];
      } else {
        return prev.includes(id) ? [] : [id];
      }
    });
  };
  
  return (
    <AccordionContext.Provider value={{ openItems, toggleItem }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ id, children }) {
  return <div className="accordion-item">{children}</div>;
}

function AccordionHeader({ id, children }) {
  const { openItems, toggleItem } = useContext(AccordionContext);
  const isOpen = openItems.includes(id);
  
  return (
    <button
      className="accordion-header"
      onClick={() => toggleItem(id)}
    >
      {children}
      <span>{isOpen ? '▼' : '▶'}</span>
    </button>
  );
}

function AccordionPanel({ id, children }) {
  const { openItems } = useContext(AccordionContext);
  const isOpen = openItems.includes(id);
  
  if (!isOpen) return null;
  
  return <div className="accordion-panel">{children}</div>;
}

Accordion.Item = AccordionItem;
Accordion.Header = AccordionHeader;
Accordion.Panel = AccordionPanel;

// Usage
<Accordion allowMultiple>
  <Accordion.Item id="1">
    <Accordion.Header id="1">Section 1</Accordion.Header>
    <Accordion.Panel id="1">Content 1</Accordion.Panel>
  </Accordion.Item>
  
  <Accordion.Item id="2">
    <Accordion.Header id="2">Section 2</Accordion.Header>
    <Accordion.Panel id="2">Content 2</Accordion.Panel>
  </Accordion.Item>
</Accordion>
```

---

### When to Use

✅ **Use when:**
- Building component libraries (UI kits)
- Complex UI với nhiều variants (tabs, accordion, dropdown)
- Cần flexible composition
- State cần share giữa nhiều children

❌ **Don't use when:**
- Simple components (overkill)
- State không cần share
- Prefer explicit props (easier to understand)

---

## 2. Render Props Pattern (Legacy)

### Concept

> **Render Prop = Function as children/prop để share logic.**

**Before Hooks, this was THE way to share stateful logic.**

---

### Basic Implementation

```jsx
// Pattern 1: Function as children
class MouseTracker extends React.Component {
  state = { x: 0, y: 0 };
  
  handleMouseMove = (e) => {
    this.setState({ x: e.clientX, y: e.clientY });
  };
  
  componentDidMount() {
    window.addEventListener('mousemove', this.handleMouseMove);
  }
  
  componentWillUnmount() {
    window.removeEventListener('mousemove', this.handleMouseMove);
  }
  
  render() {
    return this.props.children(this.state);
  }
}

// Usage
<MouseTracker>
  {({ x, y }) => (
    <div>
      Mouse position: {x}, {y}
    </div>
  )}
</MouseTracker>
```

---

### Pattern 2: Named Render Prop

```jsx
class DataFetcher extends React.Component {
  state = { data: null, loading: true, error: null };
  
  async componentDidMount() {
    try {
      const response = await fetch(this.props.url);
      const data = await response.json();
      this.setState({ data, loading: false });
    } catch (error) {
      this.setState({ error, loading: false });
    }
  }
  
  render() {
    return this.props.render(this.state);
  }
}

// Usage
<DataFetcher
  url="/api/posts"
  render={({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <Error error={error} />;
    return <PostList posts={data} />;
  }}
/>
```

---

### Modern Equivalent: Custom Hook

```jsx
// ✅ Modern way (same functionality, cleaner)
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMouseMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);
  
  return position;
}

// Usage (much simpler!)
function Component() {
  const { x, y } = useMousePosition();
  return <div>Mouse: {x}, {y}</div>;
}
```

---

### When to Use (Now)

🚫 **Don't use for new code**
- Hooks are simpler and more composable
- Render props = class components = legacy

✅ **Know it for:**
- Reading legacy codebases
- Understanding old libraries (React Motion, Downshift v1)
- Migration planning

---

## 3. Higher-Order Components (HOC) - Legacy

### Concept

> **HOC = Function that takes a component, returns enhanced component.**

**Pattern from functional programming (higher-order functions).**

---

### Basic Implementation

```jsx
// HOC: Add loading state to any component
function withLoading(Component) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) {
      return <Spinner />;
    }
    return <Component {...props} />;
  };
}

// Original component
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}

// Enhanced component
const UserListWithLoading = withLoading(UserList);

// Usage
<UserListWithLoading users={users} isLoading={loading} />
```

---

### Real Example: Authentication HOC

```jsx
function withAuth(Component) {
  return function WithAuthComponent(props) {
    const { user, loading } = useAuth(); // Modern: use hooks inside HOC
    
    if (loading) {
      return <Spinner />;
    }
    
    if (!user) {
      return <Navigate to="/login" />;
    }
    
    return <Component {...props} user={user} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(Dashboard);
```

---

### Multiple HOCs (Composition)

```jsx
// ❌ Nested hell
const EnhancedComponent = withAuth(
  withLoading(
    withErrorBoundary(
      withAnalytics(Component)
    )
  )
);

// ✅ Better: compose utility
import { compose } from 'redux'; // or lodash/fp

const enhance = compose(
  withAuth,
  withLoading,
  withErrorBoundary,
  withAnalytics
);

const EnhancedComponent = enhance(Component);
```

---

### Problems with HOCs

```jsx
// ❌ Problem 1: Props collision
function withUser(Component) {
  return function(props) {
    return <Component {...props} name="John" />;
  };
}

function withAdmin(Component) {
  return function(props) {
    return <Component {...props} name="Admin" />; // Collision!
  };
}

// ❌ Problem 2: Wrapper hell (DevTools)
<WithAuth>
  <WithLoading>
    <WithErrorBoundary>
      <Component />
    </WithErrorBoundary>
  </WithLoading>
</WithAuth>

// ❌ Problem 3: Ref không forward được
const EnhancedComponent = withSomething(Component);
<EnhancedComponent ref={myRef} /> // Ref points to HOC, not Component!
```

---

### Modern Equivalent: Custom Hooks + Components

```jsx
// ✅ Modern way
function ProtectedRoute({ children }) {
  const { user, loading } = useAuth();
  
  if (loading) return <Spinner />;
  if (!user) return <Navigate to="/login" />;
  
  return children;
}

// Usage (cleaner)
<ProtectedRoute>
  <Dashboard />
</ProtectedRoute>
```

---

### When to Use (Now)

🚫 **Don't use for new code**
- Hooks replace 99% of HOC use cases
- Complicated mental model

✅ **Know it for:**
- Legacy Redux (`connect()`)
- Old React Router (`withRouter()`)
- Legacy libraries
- Understanding `React.memo()` (technically a HOC)

---

## 4. Hooks Composition Pattern (Modern)

### Concept

> **Custom hooks = Reusable stateful logic without wrapper components.**

**This is THE modern way to share logic.**

---

### Basic Custom Hook

```jsx
// Simple custom hook
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  
  const toggle = useCallback(() => {
    setValue(v => !v);
  }, []);
  
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);
  
  return { value, toggle, setTrue, setFalse };
}

// Usage
function Modal() {
  const { value: isOpen, toggle, setFalse } = useToggle();
  
  return (
    <>
      <button onClick={toggle}>Open Modal</button>
      {isOpen && (
        <ModalContent onClose={setFalse} />
      )}
    </>
  );
}
```

---

### Composing Multiple Hooks

```jsx
// Hook 1: Fetch data
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(error => {
        if (!cancelled) {
          setError(error);
          setLoading(false);
        }
      });
    
    return () => { cancelled = true; };
  }, [url]);
  
  return { data, loading, error };
}

// Hook 2: Pagination state
function usePagination(initialPage = 1) {
  const [page, setPage] = useState(initialPage);
  
  const nextPage = useCallback(() => setPage(p => p + 1), []);
  const prevPage = useCallback(() => setPage(p => Math.max(1, p - 1)), []);
  const goToPage = useCallback((p) => setPage(p), []);
  
  return { page, nextPage, prevPage, goToPage };
}

// Hook 3: Compose both
function usePaginatedFetch(baseUrl) {
  const { page, nextPage, prevPage, goToPage } = usePagination();
  const url = `${baseUrl}?page=${page}`;
  const { data, loading, error } = useFetch(url);
  
  return {
    data,
    loading,
    error,
    page,
    nextPage,
    prevPage,
    goToPage
  };
}

// Usage
function PostList() {
  const { data, loading, page, nextPage, prevPage } = 
    usePaginatedFetch('/api/posts');
  
  if (loading) return <Spinner />;
  
  return (
    <>
      <Posts posts={data} />
      <button onClick={prevPage}>Previous</button>
      <span>Page {page}</span>
      <button onClick={nextPage}>Next</button>
    </>
  );
}
```

---

### Advanced: Hook with Reducer

```jsx
function useAsync(asyncFunction) {
  const [state, dispatch] = useReducer(
    (state, action) => {
      switch (action.type) {
        case 'pending':
          return { status: 'pending', data: null, error: null };
        case 'resolved':
          return { status: 'resolved', data: action.data, error: null };
        case 'rejected':
          return { status: 'rejected', data: null, error: action.error };
        default:
          throw new Error(`Unhandled action: ${action.type}`);
      }
    },
    { status: 'idle', data: null, error: null }
  );
  
  const run = useCallback((params) => {
    dispatch({ type: 'pending' });
    asyncFunction(params)
      .then(data => dispatch({ type: 'resolved', data }))
      .catch(error => dispatch({ type: 'rejected', error }));
  }, [asyncFunction]);
  
  return { ...state, run };
}

// Usage
function UserProfile({ userId }) {
  const { data, status, error, run } = useAsync(fetchUser);
  
  useEffect(() => {
    run(userId);
  }, [userId, run]);
  
  if (status === 'pending') return <Spinner />;
  if (status === 'rejected') return <Error error={error} />;
  if (status === 'resolved') return <Profile user={data} />;
  
  return null;
}
```

---

### Hooks Composition Best Practices

```jsx
// ✅ DO: Single Responsibility
function useAuth() { /* only auth logic */ }
function useFetch() { /* only fetching */ }
function usePagination() { /* only pagination */ }

// ❌ DON'T: God Hook
function useEverything() {
  // auth + fetch + pagination + analytics + ...
  // Too complex, hard to test, hard to reuse
}

// ✅ DO: Compose small hooks
function useAuthenticatedFetch(url) {
  const { token } = useAuth();
  const { data, loading } = useFetch(url, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return { data, loading };
}

// ✅ DO: Return object (not array) for >2 values
function useForm() {
  return {
    values,
    errors,
    handleChange,
    handleSubmit,
    reset
  };
}

// ❌ DON'T: Array for many values (confusing order)
function useForm() {
  return [values, errors, handleChange, handleSubmit, reset];
}
```

---

### When to Use

✅ **Always prefer custom hooks for:**
- Sharing stateful logic
- Side effects (fetch, timers, subscriptions)
- Complex state management (useReducer)
- Abstracting browser APIs (localStorage, geolocation)

---

## 5. Controlled vs Uncontrolled Components

### Concept

**Controlled:** React controls the value (single source of truth = state)  
**Uncontrolled:** DOM controls the value (access via ref)

---

### Uncontrolled Component

```jsx
function UncontrolledForm() {
  const nameRef = useRef();
  const emailRef = useRef();
  
  const handleSubmit = (e) => {
    e.preventDefault();
    
    // Read values from DOM
    const name = nameRef.current.value;
    const email = emailRef.current.value;
    
    console.log({ name, email });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        ref={nameRef}
        defaultValue="John" // defaultValue, not value
      />
      <input
        type="email"
        ref={emailRef}
        defaultValue="john@example.com"
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Pros:**
- Less code
- Better performance (no re-renders on change)
- React không control → DOM faster

**Cons:**
- Harder to validate
- Can't transform input (uppercase, format)
- Can't conditionally disable submit
- Harder to test

---

### Controlled Component

```jsx
function ControlledForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [errors, setErrors] = useState({});
  
  const validate = () => {
    const newErrors = {};
    if (!name) newErrors.name = 'Name required';
    if (!email) newErrors.email = 'Email required';
    if (email && !email.includes('@')) newErrors.email = 'Invalid email';
    return newErrors;
  };
  
  const handleSubmit = (e) => {
    e.preventDefault();
    
    const newErrors = validate();
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }
    
    console.log({ name, email });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={name} // value, not defaultValue
        onChange={(e) => setName(e.target.value)}
      />
      {errors.name && <span className="error">{errors.name}</span>}
      
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      {errors.email && <span className="error">{errors.email}</span>}
      
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Pros:**
- Full control (validate, transform, disable)
- Can derive state (e.g., "Submit" disabled if invalid)
- Easier to test (just check state)
- Can sync with other components

**Cons:**
- More code
- Re-render on every keystroke
- Can have performance issues (large forms)

---

### Hybrid Approach: Controlled with Performance

```jsx
import { useDeferredValue } from 'react';

function SearchInput() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // Expensive search
  const results = useMemo(() => {
    return searchData(deferredQuery);
  }, [deferredQuery]);
  
  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        // Input updates immediately (responsive)
      />
      <Results data={results} />
      {/* Results update after input stops (performance) */}
    </>
  );
}
```

---

### When to Use Each

| Use Case | Controlled | Uncontrolled |
|----------|-----------|--------------|
| Simple form (just submit) | ❌ | ✅ |
| Need validation | ✅ | ❌ |
| Transform input (uppercase) | ✅ | ❌ |
| Conditional UI (disable button) | ✅ | ❌ |
| Large form (performance) | ⚠️ (use debounce) | ✅ |
| File input | ❌ | ✅ (must use ref) |
| Integration with non-React | ❌ | ✅ |

---

### File Input (Always Uncontrolled)

```jsx
function FileUpload() {
  const fileRef = useRef();
  
  const handleSubmit = (e) => {
    e.preventDefault();
    
    const file = fileRef.current.files[0];
    console.log(file);
    
    // Upload file...
  };
  
  return (
    <form onSubmit={handleSubmit}>
      {/* File inputs are ALWAYS uncontrolled */}
      <input type="file" ref={fileRef} />
      <button type="submit">Upload</button>
    </form>
  );
}
```

---

## 6. Pattern Decision Matrix

| Pattern | Use Case | Modern? | Complexity |
|---------|----------|---------|------------|
| **Compound Components** | UI libraries, flexible composition | ✅ Yes | ⭐⭐⭐ Medium |
| **Render Props** | Legacy codebases | ❌ No | ⭐⭐⭐ Medium |
| **HOC** | Legacy codebases, `connect()`, `withRouter()` | ❌ No | ⭐⭐⭐⭐ Hard |
| **Hooks Composition** | Share logic, side effects | ✅ Yes | ⭐⭐ Easy |
| **Controlled** | Forms with validation | ✅ Default | ⭐⭐ Easy |
| **Uncontrolled** | Simple forms, file inputs | ✅ Sometimes | ⭐ Easy |

---

## 7. Migration Guide

### From Render Props → Custom Hook

```jsx
// ❌ Old (Render Props)
<MouseTracker>
  {({ x, y }) => <div>Mouse: {x}, {y}</div>}
</MouseTracker>

// ✅ New (Custom Hook)
function Component() {
  const { x, y } = useMousePosition();
  return <div>Mouse: {x}, {y}</div>;
}
```

---

### From HOC → Hooks + Component

```jsx
// ❌ Old (HOC)
const ProtectedDashboard = withAuth(Dashboard);

// ✅ New (Component + Hook)
function ProtectedRoute({ children }) {
  const { user, loading } = useAuth();
  if (loading) return <Spinner />;
  if (!user) return <Navigate to="/login" />;
  return children;
}

<ProtectedRoute>
  <Dashboard />
</ProtectedRoute>
```

---

## Tổng Kết

### Modern React Patterns (2024+)

**✅ Use these:**
1. **Custom Hooks** (sharing logic)
2. **Compound Components** (UI libraries)
3. **Controlled Components** (forms với validation)
4. **Uncontrolled** (file inputs, simple forms)

**🚫 Avoid these (legacy):**
1. Render Props → Use custom hooks
2. HOCs → Use custom hooks + components

### Key Principles

```
1. Hooks First
   → Default cho mọi reusable logic

2. Composition Over Inheritance
   → Compound components, not complex hierarchies

3. Controlled by Default
   → Uncontrolled chỉ khi performance critical

4. Keep It Simple
   → Don't over-engineer, choose simplest pattern that works
```

---

**Next:** Practice implementing these patterns in real projects! 🚀
