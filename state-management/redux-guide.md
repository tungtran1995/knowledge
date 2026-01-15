# Redux Guide – Enterprise State Management

> **When to use:** Complex apps, many features, need DevTools/middleware, team familiar with Redux.  
> **Modern approach:** Redux Toolkit (RTK) - the official recommended way.

---

## 0. Redux in 2024: Still Relevant?

### Current State

**✅ Use Redux when:**
- Large team (consistency, predictability)
- Complex state logic (middleware, time-travel debugging)
- Need DevTools (critical for debugging)
- Existing Redux codebase

**❌ Don't use Redux when:**
- Small app (<10K LOC)
- Team unfamiliar (use Zustand instead)
- Just need global theme (Context enough)
- Mostly server state (use React Query)

**Reality:** Redux usage declining but still ~40% of React apps in 2024.

---

## 1. Redux Core Concepts

### The Flow

```
Action → Dispatch → Reducer → Store → UI
   ↑                                    ↓
   └────────────────────────────────────┘
```

**Example:**
```javascript
// 1. Action: What happened
{ type: 'counter/increment' }

// 2. Dispatch: Send action to store
dispatch({ type: 'counter/increment' });

// 3. Reducer: How state changes
function counterReducer(state = 0, action) {
  switch (action.type) {
    case 'counter/increment':
      return state + 1;
    default:
      return state;
  }
}

// 4. Store: Holds state
const store = createStore(counterReducer);

// 5. UI: Selects state
const count = useSelector(state => state.counter);
```

---

### Three Principles

**1. Single source of truth**
```javascript
// ❌ Multiple stores
const userStore = createStore(userReducer);
const postsStore = createStore(postsReducer);

// ✅ One store with multiple slices
const store = createStore({
  user: userReducer,
  posts: postsReducer
});
```

**2. State is read-only**
```javascript
// ❌ Mutate state
state.count++;

// ✅ Return new state
return { ...state, count: state.count + 1 };
```

**3. Changes made with pure functions (reducers)**
```javascript
// ✅ Pure reducer
function counterReducer(state = 0, action) {
  return state + 1; // No side effects
}

// ❌ Impure (side effects)
function badReducer(state, action) {
  fetch('/api/log'); // Side effect!
  return state + 1;
}
```

---

## 2. Redux Toolkit (Modern Way)

### Why Redux Toolkit?

**Old Redux problems:**
- Too much boilerplate
- Manual immutable updates
- No built-in async handling
- Complex setup

**Redux Toolkit solutions:**
- `createSlice` (less boilerplate)
- Immer for immutability
- `createAsyncThunk` for async
- Auto DevTools setup

---

### Basic Setup

```javascript
// store.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';

const store = configureStore({
  reducer: {
    counter: counterReducer
  }
});

export default store;
```

```javascript
// App.jsx
import { Provider } from 'react-redux';
import store from './store';

function App() {
  return (
    <Provider store={store}>
      <YourApp />
    </Provider>
  );
}
```

---

### createSlice (Core API)

```javascript
// counterSlice.js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => {
      // Looks like mutation, but Immer handles immutability
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action) => {
      state.value += action.payload;
    }
  }
});

// Auto-generated actions
export const { increment, decrement, incrementByAmount } = counterSlice.actions;

// Reducer
export default counterSlice.reducer;
```

**Usage in component:**
```javascript
import { useSelector, useDispatch } from 'react-redux';
import { increment, decrement } from './counterSlice';

function Counter() {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
    </div>
  );
}
```

---

### Async Logic with createAsyncThunk

```javascript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// Async thunk
export const fetchPosts = createAsyncThunk(
  'posts/fetchPosts',
  async (userId) => {
    const response = await fetch(`/api/users/${userId}/posts`);
    return response.json();
  }
);

const postsSlice = createSlice({
  name: 'posts',
  initialState: {
    items: [],
    status: 'idle', // 'idle' | 'loading' | 'succeeded' | 'failed'
    error: null
  },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchPosts.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(fetchPosts.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchPosts.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message;
      });
  }
});

export default postsSlice.reducer;
```

**Usage:**
```javascript
function Posts() {
  const dispatch = useDispatch();
  const posts = useSelector(state => state.posts.items);
  const status = useSelector(state => state.posts.status);
  
  useEffect(() => {
    if (status === 'idle') {
      dispatch(fetchPosts(userId));
    }
  }, [status, dispatch]);
  
  if (status === 'loading') return <Spinner />;
  if (status === 'failed') return <Error />;
  
  return <PostsList posts={posts} />;
}
```

---

## 3. Selectors & Memoization

### Basic Selectors

```javascript
// ❌ Inline selector (new function every render)
function Counter() {
  const count = useSelector(state => state.counter.value);
}

// ✅ Extract selector
const selectCount = (state) => state.counter.value;

function Counter() {
  const count = useSelector(selectCount);
}
```

---

### Reselect (Memoized Selectors)

```javascript
import { createSelector } from '@reduxjs/toolkit';

// Input selector
const selectTodos = (state) => state.todos.items;

// Memoized selector
const selectCompletedTodos = createSelector(
  [selectTodos],
  (todos) => todos.filter(todo => todo.completed)
);

// Only recomputes when todos change
function CompletedTodos() {
  const completed = useSelector(selectCompletedTodos);
  return <TodoList todos={completed} />;
}
```

**Multiple inputs:**
```javascript
const selectTodoById = createSelector(
  [selectTodos, (state, todoId) => todoId],
  (todos, todoId) => todos.find(todo => todo.id === todoId)
);

// Usage
const todo = useSelector(state => selectTodoById(state, todoId));
```

---

## 4. RTK Query (Server State in Redux)

### Setup

```javascript
// services/api.js
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    getPosts: builder.query({
      query: () => 'posts'
    }),
    getPost: builder.query({
      query: (id) => `posts/${id}`
    }),
    createPost: builder.mutation({
      query: (post) => ({
        url: 'posts',
        method: 'POST',
        body: post
      })
    }),
    updatePost: builder.mutation({
      query: ({ id, ...updates }) => ({
        url: `posts/${id}`,
        method: 'PUT',
        body: updates
      })
    })
  })
});

export const {
  useGetPostsQuery,
  useGetPostQuery,
  useCreatePostMutation,
  useUpdatePostMutation
} = api;
```

**Add to store:**
```javascript
import { configureStore } from '@reduxjs/toolkit';
import { api } from './services/api';

const store = configureStore({
  reducer: {
    [api.reducerPath]: api.reducer
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(api.middleware)
});
```

---

### Usage

```javascript
function Posts() {
  const { data, error, isLoading } = useGetPostsQuery();
  
  if (isLoading) return <Spinner />;
  if (error) return <Error />;
  
  return <PostsList posts={data} />;
}

function CreatePost() {
  const [createPost, { isLoading }] = useCreatePostMutation();
  
  const handleSubmit = async (post) => {
    await createPost(post).unwrap();
  };
  
  return <PostForm onSubmit={handleSubmit} loading={isLoading} />;
}
```

---

### Cache Invalidation

```javascript
export const api = createApi({
  // ...
  tagTypes: ['Post'],
  endpoints: (builder) => ({
    getPosts: builder.query({
      query: () => 'posts',
      providesTags: ['Post'] // This query provides 'Post' data
    }),
    createPost: builder.mutation({
      query: (post) => ({
        url: 'posts',
        method: 'POST',
        body: post
      }),
      invalidatesTags: ['Post'] // Invalidate 'Post' queries after mutation
    })
  })
});

// When createPost succeeds, getPosts refetches automatically
```

---

## 5. Middleware

### Concept

**Middleware = Code that runs between dispatch and reducer**

```
dispatch(action) → Middleware → Reducer → Store
                      ↓
                   (Logging, API calls, etc.)
```

---

### Logger Middleware (Example)

```javascript
const logger = (store) => (next) => (action) => {
  console.log('Dispatching:', action);
  const result = next(action);
  console.log('Next state:', store.getState());
  return result;
};

const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(logger)
});
```

---

### Redux Thunk (Built-in RTK)

```javascript
// Thunk: Function that returns function
const fetchUser = (userId) => {
  return async (dispatch, getState) => {
    dispatch({ type: 'user/fetchStart' });
    
    try {
      const response = await fetch(`/api/users/${userId}`);
      const user = await response.json();
      dispatch({ type: 'user/fetchSuccess', payload: user });
    } catch (error) {
      dispatch({ type: 'user/fetchFailure', payload: error.message });
    }
  };
};

// Dispatch thunk
dispatch(fetchUser(123));
```

**Modern way: `createAsyncThunk` (covered earlier)** - preferred.

---

## 6. Redux DevTools

### Features

**1. Time-travel debugging**
```
Click any action in history → See state at that point
```

**2. Action inspection**
```
See all dispatched actions
View payload
Jump to action
```

**3. State diff**
```
Compare before/after state for each action
```

**4. Test action generators**
```
Dispatch custom actions manually to test
```

---

### Setup (RTK auto-configured)

```javascript
const store = configureStore({
  reducer: rootReducer
  // DevTools enabled by default in development
});
```

**Manual control:**
```javascript
const store = configureStore({
  reducer: rootReducer,
  devTools: process.env.NODE_ENV !== 'production'
});
```

---

## 7. Redux vs Zustand

| Feature | Redux Toolkit | Zustand |
|---------|--------------|---------|
| **Boilerplate** | Medium (slices) | Minimal |
| **DevTools** | ✅ Excellent | ✅ Yes (with middleware) |
| **Middleware** | ✅ Rich ecosystem | ⚠️ Limited |
| **Learning Curve** | Steep | Easy |
| **TypeScript** | ✅ Excellent | ✅ Good |
| **Async** | `createAsyncThunk`, RTK Query | Manual |
| **Bundle Size** | ~10KB | ~1KB |
| **Time-travel** | ✅ Built-in | ❌ No |
| **Immer** | ✅ Built-in | ✅ Optional |

**Decision:**
- **Redux:** Large team, complex app, need DevTools/middleware
- **Zustand:** Small team, simpler app, want minimal API

---

## 8. Migration Patterns

### From Context to Redux

**Before (Context):**
```javascript
const TodoContext = createContext();

function TodoProvider({ children }) {
  const [todos, setTodos] = useState([]);
  const addTodo = (todo) => setTodos([...todos, todo]);
  
  return (
    <TodoContext.Provider value={{ todos, addTodo }}>
      {children}
    </TodoContext.Provider>
  );
}
```

**After (Redux):**
```javascript
const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => {
      state.push(action.payload);
    }
  }
});

// No Provider in component tree (Provider at root only)
```

---

### From Redux to React Query (Server State)

**Before (Redux for API data):**
```javascript
const postsSlice = createSlice({
  name: 'posts',
  initialState: { items: [], loading: false },
  reducers: {
    fetchStart: (state) => { state.loading = true; },
    fetchSuccess: (state, action) => {
      state.items = action.payload;
      state.loading = false;
    }
  }
});

// Thunk
export const fetchPosts = () => async (dispatch) => {
  dispatch(fetchStart());
  const data = await fetch('/api/posts').then(r => r.json());
  dispatch(fetchSuccess(data));
};
```

**After (React Query):**
```javascript
// Just this
function Posts() {
  const { data, isLoading } = useQuery('posts', fetchPosts);
}
```

**Result:** 80% less code for server state.

---

## 9. Best Practices

### ✅ Do

**1. Normalize state shape**
```javascript
// ✅ Normalized
{
  users: {
    byId: {
      '1': { id: '1', name: 'Alice' },
      '2': { id: '2', name: 'Bob' }
    },
    allIds: ['1', '2']
  }
}

// ❌ Nested / duplicated
{
  posts: [
    {
      id: 1,
      author: { id: '1', name: 'Alice' } // Duplicated user data
    }
  ]
}
```

**2. Split slices by feature**
```javascript
store: {
  auth: authSlice,
  posts: postsSlice,
  comments: commentsSlice
}
```

**3. Use selectors**
```javascript
// Extract to selectors file
export const selectUser = (state) => state.auth.user;
export const selectIsAuthenticated = (state) => !!state.auth.user;
```

---

### ❌ Don't

**1. Put everything in Redux**
```javascript
// ❌ Modal state in Redux
dispatch(openModal('confirm'));

// ✅ Local state
const [isOpen, setIsOpen] = useState(false);
```

**2. Mutate state directly**
```javascript
// ❌ Even with RTK (outside reducer)
const state = store.getState();
state.count++; // Never do this

// ✅ Always dispatch actions
dispatch(increment());
```

**3. Access store directly in components**
```javascript
// ❌ Import store
import store from './store';
const count = store.getState().counter;

// ✅ Use hooks
const count = useSelector(state => state.counter);
```

---

## 10. Tổng kết Redux

### When to Choose Redux

**✅ Redux is right if:**
- App > 10K LOC
- Team > 5 developers
- Need time-travel debugging
- Complex state interactions
- Many async operations
- Existing Redux codebase

**❌ Reconsider if:**
- Small app
- Team learning React
- Mostly server state (use React Query)
- Want faster iteration

---

### Modern Redux Stack

```
Redux Toolkit (core)
+ RTK Query (server state, optional)
+ Redux DevTools
+ TypeScript (highly recommended)
```

---

### Evolution Path

```
Phase 1: Start
└─ useState + Context

Phase 2: Need Global
└─ Zustand (if simple) OR Redux (if complex)

Phase 3: Add Server State
└─ React Query or RTK Query

Phase 4: Scale
└─ Redux + React Query (separate concerns)
```

---

**COMPLETE:** You now have full state management knowledge ✅

**Next Steps:**
1. Practice with real projects
2. Read [overview.md](./overview.md) for decision framework
3. Experiment with different solutions
4. Don't over-engineer early

> **Remember:** State management is a means to an end.  
> Choose the simplest tool that solves your problem.  
> Migrate when pain points appear, not before.
