# Data Flow Architecture

> **Problem:** How should data move through your app as it grows from 1K to 100K+ LOC?

---

## 1. Flow Patterns Overview

### Unidirectional Flow (Redux-like)

```
Action → Dispatcher → Store → View
   ↑                            ↓
   └────────────────────────────┘
```

**Characteristics:**
- Data flows one direction only
- Predictable, easy to debug
- Time-travel debugging possible

**Best for:** Large teams, complex state logic

---

### Bidirectional Flow (MobX-like)

```
View ↔ Store
```

**Characteristics:**
- View can directly modify store
- Less boilerplate
- Auto-reactivity

**Best for:** Small teams, rapid prototyping

---

### Event-Driven Flow

```
ComponentA → Event Bus → ComponentB
                ↓
             ComponentC
```

**Characteristics:**
- Loose coupling
- Components don't know about each other
- Flexible, but can be hard to trace

**Best for:** Micro-frontends, plugin systems

---

## 2. Choosing the Right Pattern

### Decision Matrix

| App Size | Team Size | Complexity | Recommendation |
|----------|-----------|------------|----------------|
| Small (<10K LOC) | 1-3 | Low | Local state + Context |
| Medium (10-50K) | 3-10 | Medium | Zustand or Redux |
| Large (50K+) | 10+ | High | Redux + Domain layers |

---

## 3. Layered Architecture

### Clean Architecture for Frontend

```
┌─────────────────┐
│ Presentation    │ ← React components
├─────────────────┤
│ Application     │ ← Business logic, hooks
├─────────────────┤
│ Domain          │ ← Entities, value objects
├─────────────────┤
│ Infrastructure  │ ← API, localStorage
└─────────────────┘
```

**Dependency rule:** Inner layers don't depend on outer.

---

### Example Structure

```
src/
  presentation/
    components/     # UI components
    pages/          # Route pages
  
  application/
    hooks/          # Custom hooks (business logic)
    stores/         # State management
  
  domain/
    entities/       # User, Product, Order
    services/       # Domain logic
  
  infrastructure/
    api/            # API clients
    storage/        # localStorage, IndexedDB
```

---

## 4. Data Flow Best Practices

### Practice 1: Single Source of Truth

```javascript
// ❌ Multiple sources (inconsistent)
const user1 = await fetchUser();
const user2 = localStorage.getItem('user');
// Which one is correct?

// ✅ One source
const { data: user } = useQuery('user', fetchUser);
// Always fresh from API
```

---

### Practice 2: Immutable Updates

```javascript
// ❌ Mutation (hard to track changes)
state.user.name = 'New Name';

// ✅ Immutable (clear what changed)
setState(prev => ({
  ...prev,
  user: { ...prev.user, name: 'New Name' }
}));
```

---

### Practice 3: Separation of Concerns

```javascript
// ❌ Mixed concerns
function UserProfile() {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(setUser);
  }, []);
  
  return <div>{user?.name}</div>;
}

// ✅ Separated
function UserProfile() {
  const { data: user } = useUser(); // Data layer
  return <UserProfileView user={user} />; // Presentation layer
}
```

---

# Data Normalization - Bổ sung vào `data-flow.md`

---

## Thêm vào `data-flow.md` (sau section "Layered Architecture")

---

## 5. Data Normalization

### The Problem: Nested Duplication

```javascript
// ❌ Denormalized: Author duplicated
const posts = [
  { 
    id: 1, 
    title: 'Post 1',
    author: { id: 5, name: 'Alice', email: 'alice@example.com' }
  },
  { 
    id: 2, 
    title: 'Post 2',
    author: { id: 5, name: 'Alice', email: 'alice@example.com' } // Duplicate!
  },
  { 
    id: 3, 
    title: 'Post 3',
    author: { id: 5, name: 'Alice', email: 'alice@example.com' } // Duplicate!
  }
];

// Problem: Update Alice's email? Must update 3 places → Easy to miss
```

---

### The Solution: Normalize by ID

```javascript
// ✅ Normalized: Author stored once
const entities = {
  posts: {
    1: { id: 1, title: 'Post 1', authorId: 5 },
    2: { id: 2, title: 'Post 2', authorId: 5 },
    3: { id: 3, title: 'Post 3', authorId: 5 }
  },
  users: {
    5: { id: 5, name: 'Alice', email: 'alice@example.com' }
  }
};

// Update Alice's email? One place only
entities.users[5].email = 'newemail@example.com';

// All posts automatically get updated author
```

---

### Mental Model: Database Tables

```
Think of frontend state like database:

Posts table:
| id | title   | authorId |
|----|---------|----------|
| 1  | Post 1  | 5        |
| 2  | Post 2  | 5        |

Users table:
| id | name  | email           |
|----|-------|-----------------|
| 5  | Alice | alice@...       |

JOIN by authorId (reference, not duplication)
```

---

### When to Normalize?

**✅ Normalize when:**
```
- Same data appears multiple times
- Data gets updated frequently
- Relational data (posts, authors, comments)
- GraphQL apps (Apollo auto-normalizes)
```

**❌ Don't normalize when:**
```
- Read-only data (config, static content)
- Simple list (no relationships)
- Data fetched together always
```

---

### Implementation Patterns

**Pattern 1: Manual (Small Apps)**

```javascript
// Normalize API response
function normalizeResponse(posts) {
  const normalized = {
    posts: {},
    users: {}
  };
  
  posts.forEach(post => {
    // Store post (reference only)
    normalized.posts[post.id] = {
      id: post.id,
      title: post.title,
      authorId: post.author.id
    };
    
    // Store author (once)
    if (!normalized.users[post.author.id]) {
      normalized.users[post.author.id] = post.author;
    }
  });
  
  return normalized;
}

// Usage
const rawPosts = await api.fetchPosts();
const normalized = normalizeResponse(rawPosts);
```

---

**Pattern 2: Redux Toolkit (Auto-normalize)**

```javascript
import { createEntityAdapter } from '@reduxjs/toolkit';

// Entity adapter handles normalization
const postsAdapter = createEntityAdapter();
const usersAdapter = createEntityAdapter();

const postsSlice = createSlice({
  name: 'posts',
  initialState: postsAdapter.getInitialState(),
  reducers: {
    postsReceived: postsAdapter.setAll
  }
});

// State structure (auto-normalized):
// {
//   posts: {
//     ids: [1, 2, 3],
//     entities: {
//       1: { id: 1, title: 'Post 1', authorId: 5 },
//       2: { id: 2, title: 'Post 2', authorId: 5 }
//     }
//   }
// }

// Selectors (auto-generated)
const { selectAll, selectById } = postsAdapter.getSelectors();
```

---

**Pattern 3: Apollo Client (GraphQL)**

```javascript
import { InMemoryCache } from '@apollo/client';

// Apollo auto-normalizes by __typename + id
const cache = new InMemoryCache({
  typePolicies: {
    Post: {
      keyFields: ['id'] // Normalize by id
    },
    User: {
      keyFields: ['id']
    }
  }
});

// Query
const GET_POSTS = gql`
  query GetPosts {
    posts {
      id
      title
      author {
        id
        name
        email
      }
    }
  }
`;

// Apollo cache (auto-normalized):
// {
//   Post:1: { id: 1, title: 'Post 1', author: { __ref: 'User:5' } },
//   Post:2: { id: 2, title: 'Post 2', author: { __ref: 'User:5' } },
//   User:5: { id: 5, name: 'Alice', email: 'alice@...' }
// }

// Update one user → All posts automatically updated
```

---

### Denormalization (Rebuilding Data)

```javascript
// Get denormalized post (with author data)
function selectPostWithAuthor(state, postId) {
  const post = state.posts.entities[postId];
  const author = state.users.entities[post.authorId];
  
  return {
    ...post,
    author // Rebuild nested structure
  };
}

// Or with selectors
const selectPostWithAuthor = createSelector(
  [
    (state, postId) => state.posts.entities[postId],
    (state, postId) => state.users.entities[
      state.posts.entities[postId].authorId
    ]
  ],
  (post, author) => ({ ...post, author })
);
```

---

### Trade-offs

**Pros:**
```
✅ Single source of truth (update once)
✅ Easier updates (no need to find all duplicates)
✅ Memory efficient (no duplication)
✅ Consistent data (all references point to same object)
```

**Cons:**
```
❌ More complex queries (need JOIN logic)
❌ Extra code (normalization/denormalization)
❌ Not always needed (overkill for simple data)
```

---

### Decision Matrix

| Data Type | Normalize? | Reason |
|-----------|-----------|--------|
| Blog posts + authors | ✅ Yes | Authors repeat across posts |
| Todo items (standalone) | ❌ No | No relationships |
| E-commerce (products, orders, users) | ✅ Yes | Heavy relationships |
| Static config | ❌ No | Read-only, no updates |
| GraphQL app | ✅ Yes | Apollo auto-handles |

---

### Quick Reference

**Normalize when:**
- Same entity appears ≥3 times
- Entity gets updated
- Relational data

**Don't normalize when:**
- Data fetched once, never updated
- Simple list, no relationships
- Small app (<10K LOC)

**Rule of thumb:** If you're updating nested data and thinking "I need to update this in 5 places", time to normalize.

---

## Thêm Migration Strategies vào `architecture-decisions.md`

---

## 9. Migration Strategies (thêm sau section "Common Mistakes")

### The Migration Problem

```
Reality: Can't rewrite from scratch
- Business needs features during migration
- Risk of breaking existing functionality
- Must maintain both old + new during transition
```

---

### Strangler Pattern (Industry Standard)

**Concept:** Gradually replace old system

```
Month 1: New features use new pattern
Month 2: Migrate low-risk module
Month 3: Migrate another module
...
Month 12: Old pattern fully replaced
```

**Key:** Both patterns coexist during migration.

---

### Example: Redux → Zustand Migration

**Step 1: Setup coexistence**

```javascript
// Keep Redux for existing features
import { Provider } from 'react-redux';
import store from './redux/store';

// Add Zustand for new features
import { create } from 'zustand';

function App() {
  return (
    <Provider store={store}>
      <Routes />
    </Provider>
  );
}
```

**Step 2: New features use Zustand**

```javascript
// New feature: Use Zustand
const useNotificationStore = create((set) => ({
  notifications: [],
  addNotification: (n) => set((state) => ({
    notifications: [...state.notifications, n]
  }))
}));

function NewFeature() {
  const notifications = useNotificationStore(s => s.notifications);
  // ...
}
```

**Step 3: Identify low-coupling modules**

```
Priority order:
1. ✅ UI state (theme, sidebar open) - Low coupling
2. ✅ New features - No dependencies
3. ⚠️ User data - Medium coupling
4. ❌ Auth state - High coupling (last to migrate)
```

**Step 4: Migrate incrementally**

```javascript
// Week 1: Migrate theme (low risk)
const useThemeStore = create((set) => ({
  theme: 'light',
  setTheme: (theme) => set({ theme })
}));

// Remove from Redux
// Old: const theme = useSelector(state => state.ui.theme);
// New: const theme = useThemeStore(s => s.theme);
```

**Step 5: Remove Redux when done**

```javascript
// After 6 months: All features migrated
// Remove Redux completely
// - Uninstall redux, react-redux
// - Delete redux/ folder
// - Update documentation
```

---

### Common Migrations

**1. Class Components → Hooks**

```javascript
// Strangler: Keep both during transition

// Old (keep working)
class OldFeature extends Component {
  render() { /* ... */ }
}

// New features: Hooks
function NewFeature() {
  const [state, setState] = useState();
  // ...
}

// Gradually convert old components
// Priority: Most-changed files first
```

---

**2. REST → GraphQL**

```javascript
// Coexistence pattern
function App() {
  return (
    <ApolloProvider client={apolloClient}>
      {/* New features use GraphQL */}
      <NewFeature />
      
      {/* Old features keep REST */}
      <OldFeature />
    </ApolloProvider>
  );
}

// Migrate endpoint by endpoint
// - Start with read-only endpoints
// - Then mutations
// - Monitor both during transition
```

---

**3. Monolith → Micro-frontends**

```javascript
// Step 1: Extract one feature as micro-frontend
// Old monolith: dashboard, settings, reports
// New MFE: reports (standalone app)

// Step 2: Load via Module Federation
import('reports/App').then(Module => {
  // Render reports as MFE
});

// Step 3: Gradually extract more features
// Step 4: Shell app + multiple MFEs
```

---

### Migration Checklist

**Before migration:**
- ☐ Document current state (architecture, dependencies)
- ☐ Identify low-coupling modules (migrate first)
- ☐ Set up feature flags (rollback safety)
- ☐ Define success metrics (performance, bug rate)

**During migration:**
- ☐ Coexistence strategy (both patterns work)
- ☐ Incremental rollout (1 module at a time)
- ☐ Monitor metrics (catch regressions early)
- ☐ Document decisions (ADRs)

**After migration:**
- ☐ Remove old code (cleanup)
- ☐ Update documentation
- ☐ Retrospective (lessons learned)

---

### Migration Anti-patterns

**❌ Big Bang Rewrite**
```
Problem: Rewrite everything at once
Result: 6 months no features shipped → Business unhappy
Fix: Incremental migration
```

**❌ No Rollback Plan**
```
Problem: Migration breaks production
Result: Scramble to fix, can't rollback
Fix: Feature flags, gradual rollout
```

**❌ Migrate High-Coupling First**
```
Problem: Auth state migrated first (affects everything)
Result: Breaks entire app
Fix: Low-coupling modules first (theme, UI state)
```

---

### Key Insight

> **Good migration = Boring migration**

Not exciting, just steady progress. Ship features while migrating.

**Timeline:** Expect 6-12 months for large migrations (100K+ LOC).

---

## Tổng Kết

**Key Principles:**
1. Pick pattern based on scale
2. Enforce

 unidirectional flow (at scale)
3. Layer architecture (separation)
4. Single source of truth

**Next:** [Caching Strategies →](./caching.md)
