# Scalability Patterns

> **Challenge:** App works for 100 users. Will it work for 100,000? 1,000,000?

---

## 1. Performance at Scale

### Understanding Scale

**Users:**
```
100 users: Any code works
1,000 users: Need basic optimization
10,000 users: Need caching, code splitting
100,000+ users: Need CDN, edge computing
```

**Data:**
```
100 items: Render all fine
1,000 items: Need pagination
10,000 items: Need virtual scrolling
1,000,000 items: Need backend pagination + virtual scroll
```

---

## 2. Code Splitting Strategies

### Route-Based Splitting

```javascript
// Split by route (most common)
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));
const Profile = lazy(() => import('./Profile'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}
```

**Impact:** Initial bundle 500KB → 150KB (user only downloads current route).

---

### Component-Based Splitting

```javascript
// Heavy component used conditionally
const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

---

### Library Splitting

```javascript
// Split heavy libraries
const Editor = lazy(() => import(
  /* webpackChunkName: "editor" */
  './Editor' // includes Monaco Editor (2MB)
));

// Only loaded when editor page visited
```

---

## 3. Virtual Scrolling

### The Problem

```javascript
// ❌ Render 10,000 items
{items.map(item => <Item key={item.id} {...item} />)}
// 10,000 DOM nodes = Slow scroll, high memory
```

### The Solution

```javascript
// ✅ Only render visible items
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={items.length}
  itemSize={50}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      <Item {...items[index]} />
    </div>
  )}
</FixedSizeList>
```

**Result:** 10,000 DOM nodes → 20 DOM nodes (only visible).

---

## 4. Pagination vs Infinite Scroll

### Backend Pagination

```javascript
function PostsList() {
  const [page, setPage] = useState(1);
  const { data } = useQuery(['posts', page], () => fetchPosts(page));
  
  return (
    <>
      <Posts posts={data.posts} />
      <Pagination
        current={page}
        total={data.totalPages}
        onChange={setPage}
      />
    </>
  );
}
```

**Pros:** Simple, clear navigation, SEO-friendly.  
**Cons:** Extra clicks, page reloads.

---

### Infinite Scroll

```javascript
function PostsList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage
  } = useInfiniteQuery('posts', fetchPosts, {
    getNextPageParam: (lastPage) => lastPage.nextCursor
  });
  
  return (
    <InfiniteScroll
      dataLength={data?.pages.length || 0}
      next={fetchNextPage}
      hasMore={hasNextPage}
      loader={<Spinner />}
    >
      {data?.pages.map(page => (
        page.posts.map(post => <Post key={post.id} {...post} />)
      ))}
    </InfiniteScroll>
  );
}
```

**Pros:** Smooth UX, mobile-friendly.  
**Cons:** Hard to jump to specific page, SEO challenges.

---

## 5. Debouncing & Throttling

### Debounce (Wait for pause)

```javascript
// Search: Wait until user stops typing
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  
  const { data } = useQuery(['search', debouncedQuery], () => search(debouncedQuery), {
    enabled: debouncedQuery.length > 0
  });
  
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

// Every keystroke doesn't trigger API call, only after 300ms pause
```

---

### Throttle (Limit frequency)

```javascript
// Scroll: Run at most once per 100ms
function ScrollTracker() {
  const handleScroll = useThrottle(() => {
    console.log('Scroll position:', window.scrollY);
  }, 100);
  
  useEffect(() => {
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);
}

// Scroll event fires 100 times/sec, handler runs only 10 times/sec
```

---

## 6. Web Workers

### Offload Heavy Computation

```javascript
// worker.js
self.addEventListener('message', (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
});

// main.js
const worker = new Worker('worker.js');

worker.postMessage(data);

worker.onmessage = (e) => {
  setResult(e.data);
};
```

**Use for:** Image processing, data parsing, complex calculations.

---

## 7. CDN & Edge Computing

### Static Assets on CDN

```javascript
// Next.js automatic
module.exports = {
  images: {
    domains: ['cdn.example.com']
  }
};

// Images, JS, CSS served from CDN (faster globally)
```

---

### Edge Functions

```javascript
// Cloudflare Workers, Vercel Edge Functions
export default async function handler(request) {
  // Run close to user (low latency)
  const data = await fetch('/api/data');
  return new Response(data);
}
```

**Benefit:** 100ms latency → 10ms (serve from nearest edge).

---

## 8. Database-like Optimizations

### Indexing (Client-Side)

```javascript
// ❌ O(n) lookup
const user = users.find(u => u.id === userId);

// ✅ O(1) lookup
const usersById = new Map(users.map(u => [u.id, u]));
const user = usersById.get(userId);
```

---

### Normalization

```javascript
// ❌ Nested duplication
const posts = [
  { id: 1, author: { id: 1, name: 'Alice' }, ... },
  { id: 2, author: { id: 1, name: 'Alice' }, ... } // Duplicate
];

// ✅ Normalized
const state = {
  posts: { 1: { id: 1, authorId: 1 }, 2: { id: 2, authorId: 1 } },
  users: { 1: { id: 1, name: 'Alice' } }
};
```

---



## Tổng Kết

**Scalability Checklist:**

- ☐ Code splitting (routes + heavy components)
- ☐ Virtual scrolling (large lists)
- ☐ Pagination/infinite scroll (backend)
- ☐ Debounce search, throttle scroll
- ☐ Web workers (heavy computation)
- ☐ CDN (static assets)
- ☐ Normalized state (avoid duplication)

**Remember:** Optimize when needed, not prematurely. Measure first, then optimize.

**Next:** [Real-World Examples →](./examples.md)
