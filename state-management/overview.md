# State Management Overview – Decision Framework

> **Mục tiêu document:**  
> Hiểu landscape của state management, biết **khi nào dùng cái gì**, và tránh over-engineering.

---

## 0. Mental Model: State là gì?

### Định nghĩa

> **State = Data thay đổi theo thời gian và ảnh hưởng đến UI**

**Ví dụ:**
```javascript
// State examples
const [count, setCount] = useState(0);        // UI counter
const [user, setUser] = useState(null);       // User info
const [isOpen, setIsOpen] = useState(false);  // Modal state
```

**Không phải state:**
- Constants: `const API_URL = 'https://api.com'`
- Derived values: `const fullName = firstName + ' ' + lastName`
- Props: Data từ parent component

---

### State Types (Critical distinction)

```
1. CLIENT State
   - Owned by frontend
   - Examples: modal open/closed, form inputs, UI preferences
   - Lives in: Component, Context, Redux, Zustand

2. SERVER State
   - Owned by backend
   - Examples: user data, posts, todos from API
   - Lives in: React Query, SWR
   - Characteristics: Async, cacheable, can be stale
```

**90% of state management problems = mixing these two**

---

### URL State (Often Forgotten)

```
3. URL State
   - Owned by browser URL
   - Examples: filters, pagination, tabs, search query
   - Lives in: Query params, route params
   - Characteristics: Shareable, bookmarkable, browser back/forward
```

**Why URL state matters:**

```javascript
// ❌ Filter state in component
const [filter, setFilter] = useState('all');

// Problems:
// - Not shareable (can't send link to filtered view)
// - Not bookmarkable
// - Lost on refresh
// - Back button doesn't work

// ✅ Filter in URL
const [searchParams] = useSearchParams();
const filter = searchParams.get('status') || 'all';

// Benefits:
// - Shareable: /todos?status=completed
// - Bookmarkable
// - Persists on refresh
// - Browser back/forward works
```

**What belongs in URL:**
- Filters (status, category, price range)
- Pagination (page, limit)
- Sorting (sortBy, order)
- Search queries
- Selected tabs
- Modal/dialog state (если user needs to share)

**What doesn't belong in URL:**
- Form inputs (before submit)
- Modal animation state
- Hover states
- Sensitive data (passwords, tokens)

**Implementation (React Router):**

```javascript
import { useSearchParams } from 'react-router-dom';

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  const filters = {
    category: searchParams.get('category') || 'all',
    minPrice: searchParams.get('minPrice') || 0,
    page: parseInt(searchParams.get('page') || '1')
  };
  
  const updateFilter = (key, value) => {
    setSearchParams(params => {
      params.set(key, value);
      return params;
    });
  };
  
  return (
    <div>
      <select 
        value={filters.category}
        onChange={e => updateFilter('category', e.target.value)}
      >
        <option value="all">All</option>
        <option value="electronics">Electronics</option>
      </select>
      
      {/* URL updates: /products?category=electronics&page=1 */}
    </div>
  );
}
```

**Pro tip:** Combine URL state với server state (React Query)

```javascript
function Products() {
  const [searchParams] = useSearchParams();
  
  const filters = Object.fromEntries(searchParams);
  
  // React Query auto-refetches when filters change
  const { data } = useQuery(
    ['products', filters],
    () => fetchProducts(filters)
  );
}
```

---

## 1. State Management Spectrum

```
Simple ←―――――――――――――――――――――――――――――→ Complex

useState → Context → Zustand → Redux → XState
   ↓         ↓          ↓        ↓        ↓
 Local    App-wide   Global   Complex   State
 state     theme     cart     features  machines
```

**Rule of thumb:**
- Start simple (useState)
- Add complexity khi **pain points xuất hiện**
- Don't use Redux "just because"

---

## 2. The 4 Critical Questions

> **Before choosing ANY state solution, answer these 4 questions:**

### Question 1: Ai sở hữu state này?

```
1 component?
└─ useState (local state)

2-3 components (parent-children)?
└─ Props drilling (acceptable)
   OR hoist to common parent

Nhiều components cùng level?
└─ Context, Zustand, hoặc Redux

Toàn app?
└─ Context (infrequent) or Zustand/Redux (frequent)

Browser (URL)?
└─ useSearchParams, React Router
```

**Example:**
```javascript
// Modal state - only 1 component cares
const [isOpen, setIsOpen] = useState(false); // ✅ useState

// Theme - toàn app cares
const theme = useContext(ThemeContext); // ✅ Context

// Cart - nhiều pages cares
const items = useCartStore(state => state.items); // ✅ Zustand

// Filter - needs to be shareable
const filter = searchParams.get('status'); // ✅ URL
```

---

### Question 2: Tần suất thay đổi?

```
Rất ít (theme, auth user):
└─ Context API
   (Re-render toàn tree OK vì rarely changes)

Thường xuyên (cart, notifications):
└─ Zustand hoặc Redux
   (Selective subscriptions cần thiết)

Mỗi keystroke (form inputs):
└─ useState local
   (Không cần global)

Mỗi vài giây (API polling):
└─ React Query
   (Built-in refetch intervals)
```

**Rule of thumb:**
- Updates < 1/minute → Context OK
- Updates > 1/second → useState (local) or Zustand (global)

---

### Question 3: Nguồn gốc?

```
Local computation?
└─ useState, Zustand, Redux

Từ API/database?
└─ React Query hoặc SWR
   (Server state needs caching, revalidation)

Từ URL?
└─ useSearchParams
   (Browser manages, you just read)

Từ localStorage?
└─ useState + useEffect
   OR Zustand persist middleware
```

**Critical distinction:**
```javascript
// ❌ Wrong: Server data in Redux
const posts = useSelector(state => state.posts);
// Manual caching, refetching, stale handling

// ✅ Right: Server data in React Query
const { data: posts } = useQuery('posts', fetchPosts);
// Automatic caching, background refetch, stale-while-revalidate
```

---

### Question 4: Lifespan?

```
Component lifetime (unmount → lost)?
└─ useState

Route lifetime (navigate away → lost)?
└─ useState, Context scoped to route

App lifetime (refresh → lost)?
└─ Redux, Zustand, Context

Persistent (refresh → persists)?
└─ Zustand persist, localStorage, URL

Server lifetime (refresh → fresh from API)?
└─ React Query
```

**Example decisions:**
```javascript
// Form draft - should persist on refresh
const useFormStore = create(
  persist((set) => ({ draft: {} }), { name: 'form-draft' })
);

// Modal state - lost on refresh is fine
const [isOpen, setIsOpen] = useState(false);

// User data - fresh from API on refresh
const { data: user } = useQuery('user', fetchUser);
```

---

## 3. Decision Framework

### Step 1: Client State or Server State?

```
Is data from API/database?
├─ YES → Server State
│  └─ Use: React Query or SWR
│      (Handles caching, refetching, stale data)
│
└─ NO → Client State
   └─ Continue to Step 2
```

---

### Step 2: Scope of State?

```
Client state, how many components need it?

1 component
└─ Use: useState (local state)

2-3 components (parent-children)
└─ Use: Props drilling (acceptable)

Many components, same level
└─ Use: Hoist to nearest common parent
   OR Context API (if too much drilling)

Entire app (theme, auth, cart)
└─ Use: Context, Zustand, or Redux
```

---

### Step 3: Complexity Level?

```
App-wide state decided, how complex?

Simple (theme, language, single user object)
└─ Use: Context API
   Pros: Built-in, no library
   Cons: Performance issues if updates frequent

Medium (shopping cart, notifications)
└─ Use: Zustand
   Pros: Simple API, good performance
   Cons: Less tooling than Redux

Complex (many features, time-travel, middleware needed)
└─ Use: Redux Toolkit
   Pros: DevTools, middleware, predictable
   Cons: More boilerplate

Very Complex (workflows with many states/transitions)
└─ Use: XState
   Pros: Impossible states impossible
   Cons: Different mental model
```

---

## 3. Comparison Matrix

| Solution | Best For | Pros | Cons | Learning Curve |
|----------|----------|------|------|----------------|
| **useState** | Local state, 1 component | Simple, fast | Hard to share | ⭐ Easy |
| **Context** | App theme, auth, infrequent updates | Built-in, no deps | Perf issues, no DevTools | ⭐⭐ Easy |
| **Zustand** | Global cart, notifications | Minimal API, fast | Less tooling | ⭐⭐ Easy |
| **Redux** | Complex apps, many features | DevTools, middleware, ecosystem | Boilerplate | ⭐⭐⭐ Medium |
| **Jotai/Recoil** | Many derived states, fine-grained | Auto optimization | Different model | ⭐⭐⭐ Medium |
| **XState** | Complex workflows, FSM | Impossible states | Steep curve | ⭐⭐⭐⭐ Hard |
| **React Query** | Server/API data | Caching, sync, stale handling | Not for client state | ⭐⭐ Easy |

---

## 4. Common Scenarios → Recommended Solutions

### Scenario 1: Theme/Language Toggle
**Needs:** App-wide, infrequent updates, 2-3 values

**Recommended:** Context API
```javascript
const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState('dark');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <YourApp />
    </ThemeContext.Provider>
  );
}
```

**Why not Redux?** Overkill for 1 value that rarely changes.

---

### Scenario 2: Shopping Cart
**Needs:** Global, frequent updates, multiple operations (add, remove, update qty)

**Recommended:** Zustand
```javascript
import create from 'zustand';

const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ 
    items: [...state.items, item] 
  })),
  removeItem: (id) => set((state) => ({ 
    items: state.items.filter(i => i.id !== id) 
  }))
}));
```

**Why not Context?** Performance - cart items render frequently.  
**Why not Redux?** Zustand simpler cho this use case.

---

### Scenario 3: User Authentication
**Needs:** User data from API, global access, auth status

**Recommended:** React Query + Context
```javascript
// Fetch user với React Query
function useCurrentUser() {
  return useQuery('currentUser', fetchCurrentUser);
}

// Auth status trong Context
const AuthContext = createContext();

function AuthProvider({ children }) {
  const { data: user, isLoading } = useCurrentUser();
  
  return (
    <AuthContext.Provider value={{ user, isLoading }}>
      {children}
    </AuthContext.Provider>
  );
}
```

**Why hybrid?** Server state (user data) + Client state (is authenticated).

---

### Scenario 4: Complex Admin Dashboard
**Needs:** Many features, time-travel debugging, middleware (logging, analytics)

**Recommended:** Redux Toolkit
```javascript
// Redux Toolkit slice
const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => {
      state.push(action.payload);
    }
  }
});
```

**Why Redux?** DevTools for debugging, middleware ecosystem, proven at scale.

---

### Scenario 5: Multi-step Form Wizard
**Needs:** Complex workflow, validation, state transitions

**Recommended:** XState (State Machines)
```javascript
const formMachine = createMachine({
  initial: 'step1',
  states: {
    step1: {
      on: { NEXT: 'step2' }
    },
    step2: {
      on: { NEXT: 'step3', BACK: 'step1' }
    },
    step3: {
      on: { SUBMIT: 'submitted', BACK: 'step2' }
    },
    submitted: { type: 'final' }
  }
});
```

**Why XState?** Can't accidentally go from step1 → step3 (impossible states).

---

### Scenario 6: Fetching Posts/Todos from API
**Needs:** API data, caching, auto-refetch, loading states

**Recommended:** React Query
```javascript
function Posts() {
  const { data, isLoading, error } = useQuery('posts', fetchPosts);
  
  if (isLoading) return <Spinner />;
  if (error) return <Error />;
  
  return <PostList posts={data} />;
}
```

**Why React Query?** Handles caching, stale-while-revalidate, background refetch automatically.

---

## 5. Anti-Patterns (Common Mistakes)

### ❌ Anti-pattern 1: Using Redux for everything

```javascript
// ❌ Redux for modal state
dispatch(openModal('confirmDialog'));

// ✅ Local state is fine
const [isOpen, setIsOpen] = useState(false);
```

**Why wrong:** Local UI state doesn't need global store.

---

### ❌ Anti-pattern 2: Context for high-frequency updates

```javascript
// ❌ Context for form inputs (re-renders entire tree)
const FormContext = createContext();

function FormProvider({ children }) {
  const [formData, setFormData] = useState({});
  // Every keystroke → All consumers re-render
}
```

**Why wrong:** Context re-renders all consumers on any update.  
**Fix:** Use Zustand or local state + callbacks.

---

### ❌ Anti-pattern 3: React Query for client state

```javascript
// ❌ Using React Query for UI state
const { data: isModalOpen } = useQuery('modalState', () => true);

// ✅ Use useState or Zustand
const [isModalOpen, setIsModalOpen] = useState(false);
```

**Why wrong:** React Query is for server state (async, cacheable).

---

### ❌ Anti-pattern 4: Props drilling when it's painful

```javascript
// ❌ 5 levels of drilling
<App>
  <Header user={user} />
    <Nav user={user} />
      <UserMenu user={user} /> {/* Only this needs it */}
```

**Fix:** Context or global state after 3+ levels.

---

## 6. Migration Path (Evolution)

**Typical app evolution:**

```
Phase 1: MVP
└─ useState + props
   (Simple, fast to build)

Phase 2: Growing
└─ Add Context for theme/auth
   (1-2 global states)

Phase 3: Complex Features
└─ Add Zustand for cart/notifications
   (Performance matters)

Phase 4: Scale (if needed)
└─ Migrate to Redux for DevTools/middleware
   OR Add React Query for server state

Phase 5: Very Complex (rare)
└─ XState for workflows with many transitions
```

**Key insight:** You don't need all tools from Day 1.

---

## 7. Performance Considerations

### Context Performance Gotcha

```javascript
// ❌ Bad: Object/array causes re-renders
const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState('dark');
  
  // New object every render!
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <App />
    </ThemeContext.Provider>
  );
}

// ✅ Good: Memoize value
const value = useMemo(() => ({ theme, setTheme }), [theme]);
return (
  <ThemeContext.Provider value={value}>
    <App />
  </ThemeContext.Provider>
);
```

---

### When Performance Matters

**Use Zustand/Redux instead of Context if:**
- Updates > 1/second
- Many consumers (>10 components)
- Selective updates needed (only update subscribers)

**Example: Live chat**
```javascript
// Context → Every message = all components re-render
// Zustand → Only components subscribed to messages re-render
```

---

## 8. Testing Implications

**Easiest to test:**
1. useState (pure functions)
2. Zustand (direct store access)
3. Redux (time-travel, predictable)

**Harder to test:**
1. Context (need Provider wrapper)
2. XState (need to understand state machine)

**Testing example:**
```javascript
// useState - easy
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

test('increments count', () => {
  render(<Counter />);
  fireEvent.click(screen.getByRole('button'));
  expect(screen.getByRole('button')).toHaveTextContent('1');
});

// No provider needed, just render and test
```

---

## 9. Quick Decision Tree

```
START: I need to manage state

Question 1: Is it from API/database?
├─ YES → React Query (server state)
└─ NO → Continue

Question 2: How many components need it?
├─ 1 component → useState
├─ 2-3 components → Props
├─ Many components → Continue
└─

Question 3: Update frequency?
├─ Rare (theme, auth) → Context
├─ Frequent (cart, messages) → Zustand
└─

Question 4: Need DevTools/middleware?
├─ YES → Redux Toolkit
└─ NO → Zustand

Question 5: Complex workflow/FSM?
├─ YES → XState
└─ NO → Zustand or Redux
```

---

## 11. Recommended Stacks (Practical)

> **80% of apps fit into one of these 3 stacks**

### Stack A: Simple (2 Libraries)

**When:**
- Small to medium apps (90% of projects)
- Team wants simplicity
- Fast iteration priority

**Stack:**
```
React Query → Server state (ALL server data)
Zustand → Client state (global + complex UI)
```

**Why this works:**
- React Query handles 80% of state (API data)
- Zustand covers remaining client state (cart, UI, preferences)
- Only 2 mental models to learn
- Clear separation of concerns
- Both have excellent TypeScript support

**Example split:**
```javascript
// React Query: Server state
const { data: user } = useQuery('user', fetchUser);
const { data: posts } = useQuery('posts', fetchPosts);

// Zustand: Client state
const cart = useCartStore(state => state.items);
const theme = useThemeStore(state => state.theme);
```

**Bundle size:** ~15KB (React Query 12KB + Zustand 1KB)

---

### Stack B: Add XState Selectively (90% Coverage)

**When:**
- Stack A + critical workflows (checkout, payments, onboarding)
- Need to prevent impossible states
- Complex multi-step processes

**Stack:**
```
React Query → Server state
Zustand → Client state
XState → ONLY for critical workflows (checkout, payment)
```

**Why:**
- XState only where state machines shine
- Team learns XState gradually (just checkout team)
- Rest of team comfortable with RQ + Zustand
- Impossible states impossible (critical for payments)

**Example usage:**
```javascript
// 90% of app: React Query + Zustand
const { data: products } = useQuery('products', fetchProducts);
const cart = useCartStore(state => state.items);

// Checkout flow: XState
const [state, send] = useMachine(checkoutMachine);
// idle → address → payment → processing → success
// Can't go from idle → success (impossible)
```

**Bundle size:** ~30KB (RQ 12KB + Zustand 1KB + XState 17KB for checkout)

---

### Stack C: Redux-Centric (If Already Invested)

**When:**
- Team already knows Redux
- Large enterprise app
- Need time-travel debugging everywhere
- Existing Redux codebase

**Stack:**
```
Redux Toolkit → ALL client state
  ├─ RTK Query → Server state (built-in)
  └─ Redux DevTools → Debugging
XState → Critical workflows only (optional)
```

**Why:**
- Leverage existing Redux knowledge
- RTK Query comparable to React Query
- Reduce library count (1 main solution)
- DevTools for everything

**Trade-offs:**
- More boilerplate than Stack A
- Steeper learning curve for new developers
- Bundle size larger (~40KB)

**When to choose:**
- Team has 2+ years Redux experience
- Need to maintain legacy Redux code
- Enterprise with strict architecture requirements

---

### Comparison: 3 Stacks

| Factor | Stack A (RQ + Zustand) | Stack B (+ XState) | Stack C (Redux) |
|--------|----------------------|-------------------|---------------|
| **Complexity** | ⭐ Low | ⭐⭐ Medium | ⭐⭐⭐ High |
| **Bundle Size** | ~15KB | ~30KB | ~40KB |
| **Learning Curve** | Easy | Medium | Steep |
| **DevTools** | RQ DevTools | RQ + XState viz | Redux DevTools |
| **Time to Productivity** | 1-2 days | 1 week | 2-3 weeks |
| **Best For** | 80% of apps | Critical workflows | Enterprise, legacy |

**My recommendation:**
- **Start with Stack A** (React Query + Zustand)
- **Add XState** only when you hit complex workflows
- **Choose Redux** only if team already experienced or enterprise requirements

---

## 12. Top 5 Mistakes (Consolidated)

### ❌ Mistake 1: Over-Centralization

**Problem:** Everything in global store, including local UI state

```javascript
// ❌ Modal state in Redux/Zustand
const isModalOpen = useSelector(state => state.ui.isModalOpen);
dispatch(openModal());

// ✅ Local state is fine
const [isModalOpen, setIsModalOpen] = useState(false);
```

**Why wrong:**
- Unnecessary complexity
- More boilerplate
- Harder to test
- Component not reusable (depends on global store)

**Rule:** Local state by default. Global only when needed.

---

### ❌ Mistake 2: Mixing Server + Client State

**Problem:** API data in Redux/Zustand alongside UI state

```javascript
// ❌ Server state in Zustand
const useStore = create((set) => ({
  posts: [],
  loading: false,
  fetchPosts: async () => {
    set({ loading: true });
    const posts = await fetch('/api/posts').then(r => r.json());
    set({ posts, loading: false });
  }
}));

// ✅ Separate concerns
// Server state → React Query
const { data: posts, isLoading } = useQuery('posts', fetchPosts);

// Client state → Zustand
const selectedPostId = useStore(state => state.selectedPostId);
```

**Why wrong:**
- Manual caching, refetching, stale handling
- No background revalidation
- More code to maintain

**Rule:** Server state = React Query. Client state = Zustand/Redux.

---

### ❌ Mistake 3: Not Using URL State

**Problem:** Filters, pagination, tabs in component/global state

```javascript
// ❌ Filter in component state
const [filter, setFilter] = useState('all');

// Problems:
// - Can't share filtered view (send link)
// - Lost on refresh
// - Back button doesn't work

// ✅ Filter in URL
const [searchParams, setSearchParams] = useSearchParams();
const filter = searchParams.get('status') || 'all';

// Benefits:
// - Shareable: /todos?status=completed
// - Bookmarkable
// - Browser navigation works
```

**Rule:** Filters, pagination, tabs → URL (unless sensitive)

---

### ❌ Mistake 4: Premature Optimization

**Problem:** Choosing complex solution before having problem

```javascript
// ❌ Start with atomic state for 3 form fields
import { atom } from 'jotai';
const nameAtom = atom('');
const emailAtom = atom('');
const ageAtom = atom(0);

// ✅ Start simple
const [formData, setFormData] = useState({
  name: '',
  email: '',
  age: 0
});

// Add complexity later if performance issues appear
```

**Why wrong:**
- Waste time on complexity before validation
- Harder to refactor later
- Team needs to learn unnecessary concepts

**Rule:** useState → Context → Zustand → Redux (escalate when pain appears)

---

### ❌ Mistake 5: Context for High-Frequency Updates

**Problem:** Context value changes frequently, entire tree re-renders

```javascript
// ❌ Form state in Context (every keystroke)
const FormContext = createContext();

function FormProvider({ children }) {
  const [formData, setFormData] = useState({});
  // Every input change → All consumers re-render
  return (
    <FormContext.Provider value={{ formData, setFormData }}>
      {children}
    </FormContext.Provider>
  );
}

// ✅ Use Zustand instead (selective subscriptions)
const useFormStore = create((set) => ({
  formData: {},
  updateField: (field, value) => 
    set(state => ({ formData: { ...state.formData, [field]: value } }))
}));

// Or just local state if only form component needs it
```

**Rule:** Context for rare updates (<1/min). Zustand/Redux for frequent.

---

## 13. Recommendations by App Type

### Landing Page / Marketing Site
**Recommended:** useState + Context
- Minimal state needs
- No need for heavy libraries

---

### E-commerce Site
**Recommended:** React Query + Zustand
- React Query: Products, user data
- Zustand: Cart, UI state
- Why: Most state is server state

---

### Social Media / Feeds
**Recommended:** React Query + Redux Toolkit
- React Query: Posts, infinite scroll, caching
- Redux: User session, notifications
- Why: Complex client + server state

---

### Admin Dashboard
**Recommended:** Redux Toolkit + React Query
- Redux: Feature flags, user permissions, UI state
- React Query: All API data
- Why: DevTools critical for debugging

---

### SaaS App (Complex)
**Recommended:** Redux Toolkit + React Query + XState (for workflows)
- Redux: App-wide state
- React Query: Server data
- XState: Multi-step forms, approval workflows
- Why: All tools serve different purposes

---

## 11. Tổng kết

### Core Principles

**1. Match tool to problem**
- Don't use Redux for local state
- Don't use Context for high-frequency updates
- Don't use React Query for client state

**2. Start simple, add complexity when needed**
```
Pain Point → Solution
├─ Props drilling → Context
├─ Context perf issues → Zustand
├─ Need debugging → Redux
└─ Complex workflows → XState
```

**3. Separate client vs server state**
- Client state: Zustand, Redux, Context
- Server state: React Query, SWR
- Don't mix concerns

---

### Decision Summary Table

| Use Case | 1st Choice | 2nd Choice | Avoid |
|----------|-----------|------------|-------|
| Local UI state | useState | - | Redux |
| Theme/Language | Context | Zustand | Redux |
| Shopping Cart | Zustand | Redux | Context |
| Authentication | React Query + Context | Redux | useState |
| API Data | React Query | SWR | Redux |
| Complex App | Redux Toolkit | Zustand | Context |
| Workflows/FSM | XState | Redux | useState |

---

### Next Steps

1. **Read next:** `client-state.md` for useState, Context, Zustand, Atoms, XState
2. **Read next:** `server-state.md` for React Query, SWR
3. **Read next:** `redux-guide.md` for Redux Toolkit deep-dive

---

> **Final Advice:**  
> State management là về **trade-offs**, không có "best" solution.  
> Understand problem → Pick simplest tool → Evolve khi cần.  
> Over-engineering state = waste time. Under-engineering = technical debt.  
> Balance is key.

---

**NEXT:** [Client State Management →](./client-state.md)
