# Context API – Deep Understanding

> **Focus:** Hiểu bản chất Context, performance implications, và khi nào dùng/không dùng.

---

## 1. When to Use Context

### Bản Chất Context

**Context = Broadcast mechanism**
- Component ở bất kỳ đâu trong tree có thể subscribe
- **KHÔNG PHẢI** state management (chỉ là transport layer)
- Share data WITHOUT prop drilling

---

### ✅ Use Context When:

**1. Truly global data (rarely changes)**
```javascript
// Theme (changes 1-2 times per session)
const ThemeContext = createContext();

// Auth user (changes on login/logout only)
const AuthContext = createContext();

// i18n locale (changes when user switches language)
const LocaleContext = createContext();
```

**2. Dependency injection**
```javascript
// Pass services down the tree
const APIContext = createContext();
const AnalyticsContext = createContext();

// Deep components use services without prop drilling
```

**3. Avoid prop drilling (but consider alternatives first)**
```
App
└── Dashboard (doesn't use theme)
    └── Sidebar (doesn't use theme)
        └── UserMenu (needs theme)
            └── Avatar (needs theme)

Without Context: Pass theme through 3 components
With Context: Avatar directly consumes
```

---

### ❌ DON'T Use Context When:

**1. Frequent updates (form inputs, animations)**
```javascript
// ❌ BAD: Form state in Context
const FormContext = createContext();

function Input() {
  const { value, setValue } = useContext(FormContext);
  // Every keystroke → entire tree re-renders!
}

// ✅ GOOD: Local state or Zustand
const [value, setValue] = useState('');
```

**2. Data from API**
```javascript
// ❌ BAD: Server data in Context
const PostsContext = createContext();

// ✅ GOOD: React Query
const { data: posts } = useQuery('posts', fetchPosts);
```

**3. Can solve with composition**
```javascript
// ❌ BAD: Context for simple sharing
<ThemeContext.Provider value={theme}>
  <Dashboard theme={theme} />
</ThemeContext.Provider>

// ✅ GOOD: Just pass as prop
<Dashboard theme={theme} />
```

---

### Decision Tree

```
Cần share data?
├─ 1 component dùng?
│  └─ Local state (useState)
│
├─ 2-3 components (parent-children)?
│  └─ Props (hoặc component composition)
│
├─ Nhiều components, data từ API?
│  └─ React Query/SWR
│
├─ Nhiều components, frequent updates?
│  └─ Zustand/Redux
│
└─ Nhiều components, RARE updates (theme, auth)?
   └─ Context API ✅
```

---

## 2. Performance Implications

### The Problem: All Consumers Re-render

**Core issue:** Context value thay đổi → TẤT CẢ consumers re-render

```javascript
const AppContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('dark');
  const [count, setCount] = useState(0);
  
  return (
    <AppContext.Provider value={{ user, theme, count, setCount }}>
      {children}
    </AppContext.Provider>
  );
}

function ThemedButton() {
  const { theme } = useContext(AppContext);
  // ❌ Re-renders when user/count changes (even though it only needs theme)
}

function Counter() {
  const { count, setCount } = useContext(AppContext);
  // ❌ Every click re-renders ThemedButton too!
}
```

**Impact:**
- 1 context with 3 values
- 10 consumers total
- `count` updates → ALL 10 re-render (even if only 2 need count)

---

### Why This Happens

**React's Context algorithm:**
```
1. Provider value object changes? (reference equality)
   └─ YES → Mark all consumers as needing re-render
2. During render phase:
   └─ Re-render ALL consumer components
```

**Object reference matters:**
```javascript
// ❌ New object every render (even if values same)
<AppContext.Provider value={{ user, theme }}>

// Each render: value = new object
// → React sees different reference
// → All consumers re-render
```

---

### Solution 1: Split Contexts

```javascript
// ✅ Separate contexts for different concerns
const UserContext = createContext();
const ThemeContext = createContext();
const CountContext = createContext();

function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <CountProvider>
          <AppContent />
        </CountProvider>
      </ThemeProvider>
    </UserProvider>
  );
}

// Now: count updates only re-render CountContext consumers
```

**When to split:**
- Different update frequencies (theme rarely, count often)
- Different sets of consumers
- Data not related

---

### Solution 2: useMemo for Value

```javascript
function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('dark');
  
  // ✅ Memoize value object
  const value = useMemo(() => ({ user, theme, setTheme }), [user, theme]);
  
  return (
    <AppContext.Provider value={value}>
      {children}
    </AppContext.Provider>
  );
}

// Without useMemo: New object every render
// With useMemo: Same object if dependencies unchanged
```

---

### Solution 3: Separate State & Dispatch

```javascript
const StateContext = createContext();
const DispatchContext = createContext();

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <DispatchContext.Provider value={dispatch}>
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}

// Components that only dispatch don't re-render on state change
function ActionButton() {
  const dispatch = useContext(DispatchContext);
  // ✅ Doesn't re-render when state changes
}
```

---

### Solution 4: Context Selector (Manual)

```javascript
// Advanced: Manual selector
function useContextSelector(context, selector) {
  const value = useContext(context);
  const [selected, setSelected] = useState(() => selector(value));
  
  useEffect(() => {
    const newSelected = selector(value);
    if (newSelected !== selected) {
      setSelected(newSelected);
    }
  }, [value, selector, selected]);
  
  return selected;
}

// Usage
function ThemedButton() {
  const theme = useContextSelector(AppContext, state => state.theme);
  // ✅ Only re-renders when theme changes
}
```

**Note:** This is complex. Better to use Zustand if you need selectors.

---

## 3. Provider Composition

### The Wrapper Hell Problem

```javascript
// ❌ Nested hell (hard to read)
function App() {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LocaleProvider>
          <NotificationProvider>
            <AnalyticsProvider>
              <RouterProvider>
                <AppContent />
              </RouterProvider>
            </AnalyticsProvider>
          </NotificationProvider>
        </LocaleProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}
```

---

### Solution 1: Compose Utility

```javascript
// ✅ Compose multiple providers
function composeProviders(...providers) {
  return ({ children }) => {
    return providers.reduceRight(
      (child, Provider) => <Provider>{child}</Provider>,
      children
    );
  };
}

const AppProviders = composeProviders(
  AuthProvider,
  ThemeProvider,
  LocaleProvider,
  NotificationProvider,
  AnalyticsProvider
);

function App() {
  return (
    <AppProviders>
      <AppContent />
    </AppProviders>
  );
}
```

---

### Solution 2: Single Provider Component

```javascript
// ✅ Group related providers
function AppProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LocaleProvider>
          {children}
        </LocaleProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

function App() {
  return (
    <AppProviders>
      <AppContent />
    </AppProviders>
  );
}
```

---

### Provider Order Matters

```javascript
// ✅ Correct order (dependencies first)
<AuthProvider>        {/* No dependencies */}
  <ThemeProvider>     {/* Might need user preferences from auth */}
    <AppContent />
  </ThemeProvider>
</AuthProvider>

// ❌ Wrong order
<ThemeProvider>       {/* Needs auth user, but auth not ready */}
  <AuthProvider>
    <AppContent />
  </AuthProvider>
</ThemeProvider>
```

**Rule:** Dependencies first, dependents after.

---

## 4. Avoiding Re-renders

### Strategy 1: Split Contexts (Covered Above)

**Recap:**
```javascript
// ❌ 1 giant context → all consumers re-render
// ✅ Multiple small contexts → only relevant consumers re-render
```

---

### Strategy 2: Memoize Value (Covered Above)

**Recap:**
```javascript
const value = useMemo(() => ({ state, actions }), [state]);
```

---

### Strategy 3: Memoize Consumers

```javascript
// Without memo
function UserProfile() {
  const { user } = useContext(UserContext);
  return <div>{user.name}</div>;
}

// ✅ With memo (skip re-render if props same)
const UserProfile = React.memo(function UserProfile() {
  const { user } = useContext(UserContext);
  return <div>{user.name}</div>;
});

// Note: This doesn't prevent re-render from context change,
// but prevents re-render from parent re-rendering
```

---

### Strategy 4: Move Expensive Logic Outside Render

```javascript
// ❌ Expensive computation in render
function Component() {
  const { data } = useContext(DataContext);
  const processed = expensiveComputation(data); // Every re-render!
  return <div>{processed}</div>;
}

// ✅ useMemo
function Component() {
  const { data } = useContext(DataContext);
  const processed = useMemo(() => expensiveComputation(data), [data]);
  return <div>{processed}</div>;
}
```

---

### Strategy 5: Bailout with Same Value

```javascript
// React bails out if value hasn't changed
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('dark');
  
  // ✅ React checks: Is new theme === old theme?
  // If yes → Skip re-rendering consumers
  setTheme('dark'); // Same value → no re-render
}
```

---

### Strategy 6: Component Composition

```javascript
// ❌ Context wraps everything
function Dashboard() {
  return (
    <UserProvider>
      <Header />
      <Sidebar />
      <Content />
      <Footer />
    </UserProvider>
  );
}
// All 4 components re-render when user changes

// ✅ Context wraps only what needs it
function Dashboard() {
  return (
    <>
      <Header />
      <UserProvider>
        <Sidebar />
        <Content />
      </UserProvider>
      <Footer />
    </>
  );
}
// Only Sidebar + Content re-render
```

---

## 5. Common Patterns

### Pattern 1: Custom Hook

```javascript
const ThemeContext = createContext();

// Provider
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('dark');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// ✅ Custom hook with error checking
export function useTheme() {
  const context = useContext(ThemeContext);
  
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  
  return context;
}

// Usage (cleaner)
function Component() {
  const { theme, setTheme } = useTheme(); // Instead of useContext
}
```

---

### Pattern 2: Default Values

```javascript
// Context with sensible defaults
const ThemeContext = createContext({
  theme: 'light',
  setTheme: () => {
    console.warn('setTheme called outside ThemeProvider');
  }
});

// Now: Can use without Provider (for testing, Storybook)
function Button() {
  const { theme } = useContext(ThemeContext);
  // Works even without Provider (uses default)
}
```

---

### Pattern 3: Lazy Context Value

```javascript
function ExpensiveProvider({ children }) {
  // ❌ Compute on every render
  const value = computeExpensiveValue();
  
  // ✅ Lazy init (only once)
  const [value] = useState(() => computeExpensiveValue());
  
  return <Context.Provider value={value}>{children}</Context.Provider>;
}
```

---

## 6. Context vs Alternatives

| Feature | Context | Zustand | Redux | Props |
|---------|---------|---------|-------|-------|
| **Setup** | Easy | Easy | Medium | N/A |
| **Updates** | All consumers | Selective | Selective | Parent → Child |
| **Performance** | ⚠️ Poor (frequent) | ✅ Good | ✅ Good | ✅ Good |
| **DevTools** | ❌ No | ✅ Yes | ✅✅ Yes | N/A |
| **Use Case** | Theme, Auth (rare) | Global state | Complex apps | Local sharing |

---

## 7. Best Practices

### ✅ DO:

1. **Split contexts by concern**
   - UserContext, ThemeContext (not AppContext)

2. **Memoize value object**
   ```javascript
   const value = useMemo(() => ({ state }), [state]);
   ```

3. **Custom hook with error check**
   ```javascript
   if (!context) throw new Error('...');
   ```

4. **Keep context close to consumers**
   - Don't wrap entire app if only 2 components need it

5. **Use for truly global, rarely-changing data**
   - Theme, locale, auth status

---

### ❌ DON'T:

1. **Don't use for frequently changing data**
   - Form inputs → local state
   - Shopping cart → Zustand

2. **Don't use for server data**
   - Posts, users → React Query

3. **Don't create one giant context**
   - Split by concern

4. **Don't forget to memoize**
   - Prevent unnecessary re-renders

5. **Don't use when props work fine**
   - 1-2 levels → just pass props

---

## Tổng Kết

**Context API:**
- ✅ Perfect cho: Theme, Auth, i18n (rare updates)
- ⚠️ Problematic cho: Forms, frequent updates
- ❌ Wrong for: Server state, complex client state

**Key Takeaways:**
1. Context = transport, not state management
2. Performance issues = #1 gotcha
3. Split contexts to avoid re-render hell
4. Memoize value to prevent unnecessary updates
5. Consider Zustand/Redux if you need selectors

**Modern approach:**
```
Context (theme, auth)
+ React Query (server state)
+ Zustand (complex client state)
+ Local state (component state)
= Complete state management
```

---

**Next:** Practice with real examples, profile performance with React DevTools! 🚀
