# Client State Management – Deep Dive

> **Scope:** Data owned by frontend, không phải từ server.  
> **Mental model:** Client state = UI state, preferences, temporary data.

**Covered:**
1. useState - Local state
2. Context API - App-wide, infrequent updates
3. Zustand - Simple global state
4. Jotai/Recoil - Atomic state
5. XState - State machines

---

## 1. useState - The Foundation

### When to Use

**Perfect for:**
- Local component state
- Form inputs
- UI toggles (modal open/closed)
- Temporary data

**Example scenarios:**
```javascript
// Toggle
const [isOpen, setIsOpen] = useState(false);

// Form input
const [email, setEmail] = useState('');

// Counter
const [count, setCount] = useState(0);
```

---

### Best Practices

**✅ Initialize correctly:**
```javascript
// ✅ Simple value
const [count, setCount] = useState(0);

// ✅ Lazy initialization (expensive computation)
const [data, setData] = useState(() => {
  return expensiveCalculation();
});

// ❌ Don't call function directly (runs every render)
const [data, setData] = useState(expensiveCalculation());
```

---

**✅ Functional updates:**
```javascript
// ❌ May be stale
setCount(count + 1);

// ✅ Always current value
setCount(prevCount => prevCount + 1);
```

**Why functional:** Batched updates, concurrent rendering (React 18+).

---

**✅ Object/Array updates:**
```javascript
// ❌ Mutating (doesn't trigger re-render)
const [user, setUser] = useState({ name: 'Alice', age: 25 });
user.age = 26; // No re-render!

// ✅ Immutable update
setUser(prev => ({ ...prev, age: 26 }));

// Arrays
const [items, setItems] = useState([]);

// ✅ Add item
setItems(prev => [...prev, newItem]);

// ✅ Remove item
setItems(prev => prev.filter(item => item.id !== deleteId));

// ✅ Update item
setItems(prev => prev.map(item =>
  item.id === updateId ? { ...item, ...updates } : item
));
```

---

### When useState is NOT enough

**Signals:**
- Need state in 5+ components at same level
- Props drilling > 3 levels
- State updates from multiple components
- Need to share state across routes

**Example of pain:**
```javascript
// ❌ Props drilling hell
<App>
  <Header theme={theme} />
    <Nav theme={theme} />
      <Menu theme={theme} />
        <MenuItem theme={theme} /> {/* Only this needs it */}

// ✅ Move to Context or global state
```

---

## 2. Context API - Built-in Global State

### When to Use

**Perfect for:**
- App theme
- User authentication status
- Language/i18n
- Feature flags
- **Infrequent updates** (critical!)

**NOT for:**
- High-frequency updates (forms, animations)
- Large state objects that change often
- Performance-critical state

---

### Basic Pattern

```javascript
import { createContext, useContext, useState } from 'react';

// 1. Create Context
const ThemeContext = createContext();

// 2. Provider Component
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('dark');
  
  // Memoize value (prevent re-renders)
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3. Custom Hook
function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}

// 4. Usage
function App() {
  return (
    <ThemeProvider>
      <Header />
      <Main />
    </ThemeProvider>
  );
}

function Header() {
  const { theme, setTheme } = useTheme();
  
  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
      Current: {theme}
    </button>
  );
}
```

---

### Performance Optimization

**Problem: Context re-renders all consumers**

```javascript
// ❌ Every update → all consumers re-render
const AppContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('dark');
  const [notifications, setNotifications] = useState([]);
  
  return (
    <AppContext.Provider value={{ user, theme, notifications, setUser, setTheme }}>
      {children}
    </AppContext.Provider>
  );
}

// Component only needs theme, but re-renders when user/notifications change
function ThemedButton() {
  const { theme } = useContext(AppContext); // Re-renders on ANY context change
}
```

---

**✅ Solution 1: Split Contexts**

```javascript
// Separate contexts for different concerns
const UserContext = createContext();
const ThemeContext = createContext();
const NotificationContext = createContext();

function App() {
  return (
    <UserProvider>
      <ThemeProvider>
        <NotificationProvider>
          <YourApp />
        </NotificationProvider>
      </ThemeProvider>
    </UserProvider>
  );
}

// Now components only subscribe to what they need
function ThemedButton() {
  const { theme } = useTheme(); // Only re-renders on theme change
}
```

---

**✅ Solution 2: Context Selectors (Manual)**

```javascript
function useThemeSelector(selector) {
  const context = useContext(AppContext);
  const [selected, setSelected] = useState(() => selector(context));
  
  useEffect(() => {
    setSelected(selector(context));
  }, [context, selector]);
  
  return selected;
}

// Usage
function ThemedButton() {
  const theme = useThemeSelector(state => state.theme);
  // Only re-renders when theme changes
}
```

**Note:** This is complex. Better to use Zustand if you need selectors.

---

### Context Anti-Patterns

**❌ Anti-pattern 1: One giant context**

Covered above - split contexts instead.

---

**❌ Anti-pattern 2: Frequent updates**

```javascript
// ❌ Form state in Context
function FormProvider({ children }) {
  const [formData, setFormData] = useState({});
  
  return (
    <FormContext.Provider value={{ formData, setFormData }}>
      {children}
    </FormContext.Provider>
  );
}

// Every keystroke → entire tree re-renders

// ✅ Use local state or Zustand instead
```

---

**❌ Anti-pattern 3: Not memoizing value**

```javascript
// ❌ New object every render
return (
  <ThemeContext.Provider value={{ theme, setTheme }}>
    {children}
  </ThemeContext.Provider>
);

// ✅ Memoize
const value = useMemo(() => ({ theme, setTheme }), [theme]);
return (
  <ThemeContext.Provider value={value}>
    {children}
  </ThemeContext.Provider>
);
```

---

### When to Migrate Away from Context

**Signals:**
- Performance issues (use React DevTools Profiler)
- Updates > 1/second
- Many consumers (>10 components)
- Need DevTools/time-travel debugging

**Migrate to:** Zustand or Redux

---

## 3. Zustand - Minimal Global State

### Why Zustand?

**Pros:**
- Minimal API (< 1KB)
- No Provider needed
- Selective subscriptions (no unnecessary re-renders)
- Works outside React

**Cons:**
- Less tooling than Redux
- Smaller ecosystem

---

### Basic Usage

```javascript
import create from 'zustand';

// Create store
const useStore = create((set) => ({
  // State
  count: 0,
  user: null,
  
  // Actions
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  setUser: (user) => set({ user })
}));

// Usage - component re-renders ONLY when count changes
function Counter() {
  const count = useStore((state) => state.count);
  const increment = useStore((state) => state.increment);
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+</button>
    </div>
  );
}

// This component doesn't re-render when count changes
function UserProfile() {
  const user = useStore((state) => state.user);
  return <div>{user?.name}</div>;
}
```

**Key insight:** Selective subscriptions = performance.

---

### Advanced Patterns

**🔹 Immer for immutable updates**

```javascript
import create from 'zustand';
import { immer } from 'zustand/middleware/immer';

const useStore = create(
  immer((set) => ({
    todos: [],
    addTodo: (todo) => set((state) => {
      state.todos.push(todo); // Mutate with Immer (safe)
    }),
    toggleTodo: (id) => set((state) => {
      const todo = state.todos.find(t => t.id === id);
      if (todo) todo.completed = !todo.completed;
    })
  }))
);
```

---

**🔹 Persist state to localStorage**

```javascript
import create from 'zustand';
import { persist } from 'zustand/middleware';

const useStore = create(
  persist(
    (set) => ({
      theme: 'dark',
      setTheme: (theme) => set({ theme })
    }),
    {
      name: 'app-storage' // localStorage key
    }
  )
);

// State automatically saved/restored from localStorage
```

---

**🔹 DevTools integration**

```javascript
import { devtools } from 'zustand/middleware';

const useStore = create(
  devtools((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 }))
  }))
);

// Open Redux DevTools → See Zustand state
```

---

**🔹 Derived state (selectors)**

```javascript
const useStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] }))
}));

// Derived selector
const useTotal = () => {
  const items = useStore((state) => state.items);
  return items.reduce((sum, item) => sum + item.price, 0);
};

// Component
function Total() {
  const total = useTotal();
  return <div>Total: ${total}</div>;
}
```

---

### Zustand vs Context

| Feature | Context | Zustand |
|---------|---------|---------|
| API Size | Built-in | ~1KB |
| Provider needed | ✅ Yes | ❌ No |
| Selective subscriptions | ❌ Manual | ✅ Built-in |
| Performance | Poor (frequent updates) | Excellent |
| DevTools | ❌ No | ✅ Yes (with middleware) |
| Learning curve | Easy | Easy |

**Rule of thumb:** Use Context for theme. Use Zustand for cart/notifications.

---

## 4. Jotai/Recoil - Atomic State

### Concept: Atoms

**Mental model:**
```
Traditional: One big store with all state
Atomic: Many small atoms, compose as needed
```

**Example:**
```javascript
// Redux/Zustand: All in one store
{
  user: { name: 'Alice' },
  posts: [...],
  comments: [...]
}

// Jotai: Separate atoms
userAtom = { name: 'Alice' }
postsAtom = [...]
commentsAtom = [...]
```

---

### Jotai (Simpler)

```javascript
import { atom, useAtom } from 'jotai';

// Define atoms
const countAtom = atom(0);
const userAtom = atom({ name: 'Alice', age: 25 });

// Usage
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

---

**Derived atoms:**

```javascript
const todosAtom = atom([
  { id: 1, text: 'Learn Jotai', completed: false },
  { id: 2, text: 'Build app', completed: true }
]);

// Derived atom (auto-recomputes)
const completedCountAtom = atom((get) => {
  const todos = get(todosAtom);
  return todos.filter(t => t.completed).length;
});

// Usage
function Stats() {
  const [completedCount] = useAtom(completedCountAtom);
  return <p>Completed: {completedCount}</p>;
}
```

---

### When to Use Atomic State

**✅ Good for:**
- Many small, independent pieces of state
- Complex derived state (auto optimization)
- Fine-grained subscriptions

**❌ Not ideal for:**
- Simple apps (overkill)
- Team unfamiliar with concept
- Need mature ecosystem (Zustand/Redux better)

---

## 5. XState - State Machines

### Concept

**Traditional state management:**
```javascript
const [status, setStatus] = useState('idle');

// Possible states: 'idle', 'loading', 'success', 'error'
// Problem: Can accidentally do setStatus('successs') (typo)
// Problem: No constraints on transitions (idle → success?)
```

**State machine:**
```
Only allowed states exist
Only allowed transitions possible
→ Impossible states impossible
```

---

### Basic Example: Toggle

```javascript
import { createMachine, useMachine } from '@xstate/react';

const toggleMachine = createMachine({
  id: 'toggle',
  initial: 'inactive',
  states: {
    inactive: {
      on: { TOGGLE: 'active' }
    },
    active: {
      on: { TOGGLE: 'inactive' }
    }
  }
});

function Toggle() {
  const [state, send] = useMachine(toggleMachine);
  
  return (
    <button onClick={() => send('TOGGLE')}>
      {state.value}  {/* 'inactive' or 'active' */}
    </button>
  );
}
```

---

### Real Example: Fetch State Machine

```javascript
const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  context: {
    data: null,
    error: null
  },
  states: {
    idle: {
      on: { FETCH: 'loading' }
    },
    loading: {
      invoke: {
        src: 'fetchData',
        onDone: {
          target: 'success',
          actions: 'setData'
        },
        onError: {
          target: 'failure',
          actions: 'setError'
        }
      }
    },
    success: {
      on: { FETCH: 'loading' }
    },
    failure: {
      on: { RETRY: 'loading' }
    }
  }
}, {
  services: {
    fetchData: () => fetch('/api/data').then(res => res.json())
  },
  actions: {
    setData: (context, event) => {
      context.data = event.data;
    },
    setError: (context, event) => {
      context.error = event.data;
    }
  }
});

function DataFetcher() {
  const [state, send] = useMachine(fetchMachine);
  
  return (
    <div>
      {state.matches('idle') && <button onClick={() => send('FETCH')}>Load</button>}
      {state.matches('loading') && <Spinner />}
      {state.matches('success') && <Data data={state.context.data} />}
      {state.matches('failure') && (
        <div>
          <Error error={state.context.error} />
          <button onClick={() => send('RETRY')}>Retry</button>
        </div>
      )}
    </div>
  );
}
```

**Benefits:**
- Can't go from idle → success (must go through loading)
- Can't have loading + error simultaneously
- Visualizable (XState Visualizer tool)

---

### When to Use XState

**✅ Perfect for:**
- Complex workflows (multi-step forms, wizards)
- Approval flows (draft → review → approved → published)
- Game logic (menu → playing → paused → game-over)
- Authentication flows (logged-out → logging-in → logged-in)

**❌ Overkill for:**
- Simple toggles (useState is fine)
- Basic counters
- Most CRUD apps

---

## 6. Comparison & Decision Matrix

| Solution | API Complexity | Performance | DevTools | Use Case |
|----------|---------------|-------------|----------|----------|
| **useState** | ⭐ Simple | ⭐⭐⭐ Fast | ❌ | Local state |
| **Context** | ⭐⭐ Easy | ⭐ Slow (frequent updates) | ❌ | Theme, auth (infrequent) |
| **Zustand** | ⭐⭐ Easy | ⭐⭐⭐ Fast | ✅ | Cart, notifications |
| **Jotai** | ⭐⭐⭐ Medium | ⭐⭐⭐ Fast | ✅ | Fine-grained, derived state |
| **XState** | ⭐⭐⭐⭐ Complex | ⭐⭐ OK | ✅✅ Excellent | Workflows, FSM |

---

## 7. Migration Patterns

### Context → Zustand

```javascript
// Before: Context
const CartContext = createContext();

function CartProvider({ children }) {
  const [items, setItems] = useState([]);
  const addItem = (item) => setItems([...items, item]);
  
  return (
    <CartContext.Provider value={{ items, addItem }}>
      {children}
    </CartContext.Provider>
  );
}

// After: Zustand
const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] }))
}));

// Usage stays similar
function Cart() {
  const items = useCartStore((state) => state.items);
  const addItem = useCartStore((state) => state.addItem);
}
```

**Benefits:** No Provider, better performance.

---

## 8. Tổng kết Client State

### Decision Flow

```
1. Local state (1 component)?
   └─ useState

2. App-wide, infrequent changes (theme)?
   └─ Context API

3. App-wide, frequent changes (cart)?
   └─ Zustand

4. Many derived states, fine-grained?
   └─ Jotai/Recoil

5. Complex workflows, state transitions?
   └─ XState
```

### General Rules

**Start simple:**
- useState for local
- Context for 1-2 global values (theme, auth)

**Optimize when needed:**
- Context → Zustand (performance issues)
- useState → XState (complex workflows)

**Don't prematurely optimize:**
- Redux not needed for most apps under 10K LOC
- XState overkill for simple toggles
- Context fine if updates < 1/min

---

**NEXT:** [Server State Management →](./server-state.md)
