# Server State Management – React Query & SWR

> **Scope:** Data from API/database - async, cacheable, có thể stale.  
> **Key insight:** Server state ≠ Client state. Different problems, different solutions.

---

## 0. Server State vs Client State

### Fundamental Difference

**Client State:**
- Owned by frontend
- Synchronous
- Examples: modal open/closed, selected tab, user input
- Tools: useState, Zustand, Redux

**Server State:**
- Owned by backend
- Asynchronous (fetch, then render)
- Can be stale (data changes on server)
- Examples: user data, posts, products from API
- Tools: React Query, SWR

---

### Problems Server State Solves

**Without React Query/SWR:**
```javascript
// ❌ Manual management
function Posts() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetch('/api/posts')
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);
  
  // No caching, no background refetch, no stale-while-revalidate
}
```

**With React Query:**
```javascript
// ✅ Automatic caching, refetching, stale handling
function Posts() {
  const { data, isLoading, error } = useQuery('posts', fetchPosts);
  
  // That's it. Caching, background updates handled automatically.
}
```

---

## 1. React Query – Comprehensive Solution

### Core Concepts

**Query:** Fetch data
```javascript
useQuery('posts', fetchPosts);
//        ↑ key   ↑ fetcher function
```

**Mutation:** Update data (POST/PUT/DELETE)
```javascript
useMutation(createPost);
```

---

### Basic Usage

```javascript
import { useQuery } from '@tanstack/react-query';

// Fetcher function
async function fetchPosts() {
  const res = await fetch('/api/posts');
  if (!res.ok) throw new Error('Failed to fetch');
  return res.json();
}

// Component
function Posts() {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts
  });
  
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  
  return (
    <div>
      <button onClick={() => refetch()}>Refresh</button>
      {data.map(post => <Post key={post.id} {...post} />)}
    </div>
  );
}
```

---

### Query Keys (Critical Concept)

**Query key = Unique identifier for cached data**

```javascript
// Simple key
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts
});

// Key with parameters (different cache for different users)
useQuery({
  queryKey: ['posts', userId],
  queryFn: () => fetchUserPosts(userId)
});

// Key hierarchy
useQuery({
  queryKey: ['posts', 'published', { sortBy: 'date' }],
  queryFn: () => fetchPosts({ status: 'published', sortBy: 'date' })
});

// Invalidation works with hierarchy
queryClient.invalidateQueries(['posts']); // Invalidates all posts queries
queryClient.invalidateQueries(['posts', 'published']); // Only published posts
```

---

### Caching Strategy

**Default behavior:**
```javascript
useQuery({ queryKey: ['posts'], queryFn: fetchPosts });

// First time: Fetch from API → Cache → Render
// Second time (same key): Use cache → Render instantly
// Background: Refetch → Update cache if changed
```

**Stale time:**
```javascript
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  staleTime: 5 * 60 * 1000 // 5 minutes
});

// Data considered "fresh" for 5 mins
// No background refetch during this time
// After 5 mins: marked "stale", refetch in background on next render
```

**Cache time:**
```javascript
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  cacheTime: 10 * 60 * 1000 // 10 minutes
});

// Data stays in cache for 10 mins after last usage
// After 10 mins of no usage: removed from cache
```

---

### Mutations (Create/Update/Delete)

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreatePost() {
  const queryClient = useQueryClient();
  
  const mutation = useMutation({
    mutationFn: (newPost) => {
      return fetch('/api/posts', {
        method: 'POST',
        body: JSON.stringify(newPost),
        headers: { 'Content-Type': 'application/json' }
      }).then(res => res.json());
    },
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries(['posts']);
    }
  });
  
  const handleSubmit = (e) => {
    e.preventDefault();
    mutation.mutate({ title: 'New Post', body: '...' });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <button disabled={mutation.isLoading}>
        {mutation.isLoading ? 'Creating...' : 'Create Post'}
      </button>
      {mutation.isError && <Error error={mutation.error} />}
    </form>
  );
}
```

---

### Optimistic Updates

```javascript
const mutation = useMutation({
  mutationFn: updatePost,
  onMutate: async (updatedPost) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries(['posts', updatedPost.id]);
    
    // Snapshot previous value
    const previousPost = queryClient.getQueryData(['posts', updatedPost.id]);
    
    // Optimistically update cache
    queryClient.setQueryData(['posts', updatedPost.id], updatedPost);
    
    // Return snapshot for rollback
    return { previousPost };
  },
  onError: (err, updatedPost, context) => {
    // Rollback on error
    queryClient.setQueryData(
      ['posts', updatedPost.id],
      context.previousPost
    );
  },
  onSettled: (updatedPost) => {
    // Refetch to sync with server
    queryClient.invalidateQueries(['posts', updatedPost.id]);
  }
});

// Usage
function EditPost({ post }) {
  const handleSave = () => {
    mutation.mutate({ ...post, title: 'Updated' });
    // UI updates instantly (optimistic)
    // Reverts if API call fails
  };
}
```

---

### Pagination

```javascript
function Posts() {
  const [page, setPage] = useState(1);
  
  const { data, isLoading } = useQuery({
    queryKey: ['posts', page],
    queryFn: () => fetchPosts(page),
    keepPreviousData: true // Keep old data while fetching new page
  });
  
  return (
    <div>
      {data?.posts.map(post => <Post key={post.id} {...post} />)}
      
      <button onClick={() => setPage(p => p - 1)} disabled={page === 1}>
        Previous
      </button>
      
      <button onClick={() => setPage(p => p + 1)} disabled={!data?.hasMore}>
        Next
      </button>
    </div>
  );
}
```

---

### Infinite Scroll

```javascript
function InfinitePosts() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 1 }) => fetchPosts(pageParam),
    getNextPageParam: (lastPage, allPages) => {
      return lastPage.hasMore ? allPages.length + 1 : undefined;
    }
  });
  
  return (
    <div>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.posts.map(post => <Post key={post.id} {...post} />)}
        </div>
      ))}
      
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : 'Load More'}
      </button>
    </div>
  );
}
```

---

### Dependent Queries

```javascript
function UserPosts({ userId }) {
  // First query: Get user
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId)
  });
  
  // Second query: Get user's posts (only when user exists)
  const { data: posts } = useQuery({
    queryKey: ['posts', user?.id],
    queryFn: () => fetchUserPosts(user.id),
    enabled: !!user // Only run when user exists
  });
  
  return <PostsList posts={posts} />;
}
```

---

### Prefetching

```javascript
function PostsList() {
  const queryClient = useQueryClient();
  
  const { data: posts } = useQuery(['posts'], fetchPosts);
  
  return (
    <div>
      {posts.map(post => (
        <div
          key={post.id}
          onMouseEnter={() => {
            // Prefetch on hover
            queryClient.prefetchQuery({
              queryKey: ['post', post.id],
              queryFn: () => fetchPost(post.id)
            });
          }}
        >
          <Link to={`/posts/${post.id}`}>{post.title}</Link>
        </div>
      ))}
    </div>
  );
}
```

---

## 2. SWR – Alternative Solution

### Concept

**SWR = Stale-While-Revalidate**

```
1. Return stale data (cache) immediately
2. Fetch fresh data (revalidate) in background
3. Update with fresh data when ready
```

---

### Basic Usage

```javascript
import useSWR from 'swr';

const fetcher = (url) => fetch(url).then(res => res.json());

function Posts() {
  const { data, error, isLoading } = useSWR('/api/posts', fetcher);
  
  if (isLoading) return <Spinner />;
  if (error) return <Error />;
  
  return <PostsList posts={data} />;
}
```

---

### Key Features

**Auto revalidation:**
```javascript
useSWR('/api/user', fetcher, {
  revalidateOnFocus: true,       // Refetch when tab gains focus
  revalidateOnReconnect: true,   // Refetch when network reconnects
  refreshInterval: 30000          // Poll every 30 seconds
});
```

**Mutations:**
```javascript
import { mutate } from 'swr';

function CreatePost() {
  const handleCreate = async (newPost) => {
    // Optimistic update
    mutate('/api/posts', [...data, newPost], false);
    
    // Send to server
    await fetch('/api/posts', {
      method: 'POST',
      body: JSON.stringify(newPost)
    });
    
    // Revalidate
    mutate('/api/posts');
  };
}
```

---

### Pagination (SWR)

```javascript
function Posts() {
  const [pageIndex, setPageIndex] = useState(0);
  
  const { data } = useSWR(`/api/posts?page=${pageIndex}`, fetcher);
  
  return (
    <div>
      {data?.posts.map(post => <Post key={post.id} {...post} />)}
      <button onClick={() => setPageIndex(i => i + 1)}>Next</button>
    </div>
  );
}
```

---

## 3. React Query vs SWR

| Feature | React Query | SWR |
|---------|-------------|-----|
| **API Size** | ~12KB | ~4KB |
| **Mutation API** | ✅ Built-in (`useMutation`) | ⚠️ Manual (`mutate`) |
| **DevTools** | ✅ Excellent | ❌ No |
| **Optimistic Updates** | ✅ First-class | ⚠️ Manual |
| **Prefetching** | ✅ Built-in | ⚠️ Manual |
| **Infinite Queries** | ✅ `useInfiniteQuery` | ⚠️ Manual |
| **Dependent Queries** | ✅ `enabled` option | ⚠️ Conditional hook |
| **Learning Curve** | Medium | Easy |
| **Documentation** | Excellent | Good |

**Recommendation:**
- **SWR:** Simpler apps, read-heavy, small bundle size priority
- **React Query:** Complex apps, mutations, need DevTools

---

## 4. Common Patterns

### Pattern 1: Global Loading State

```javascript
import { useIsFetching } from '@tanstack/react-query';

function GlobalLoadingIndicator() {
  const isFetching = useIsFetching();
  
  return isFetching > 0 ? <TopBarLoader /> : null;
}
```

---

### Pattern 2: Error Boundary

```javascript
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ resetErrorBoundary }) => (
            <div>
              <p>Something went wrong</p>
              <button onClick={() => resetErrorBoundary()}>Try again</button>
            </div>
          )}
        >
          <Posts />
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

---

### Pattern 3: Retry Logic

```javascript
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  retry: 3,                      // Retry 3 times
  retryDelay: (attemptIndex) => 
    Math.min(1000 * 2 ** attemptIndex, 30000) // Exponential backoff
});
```

---

## 5. When NOT to Use React Query/SWR

**❌ Don't use for:**

1. **Client-only state**
   ```javascript
   // ❌ Modal state in React Query
   const { data: isOpen } = useQuery('modalState', () => true);
   
   // ✅ Just useState
   const [isOpen, setIsOpen] = useState(false);
   ```

2. **State that never comes from server**
   ```javascript
   // ❌ Form input in React Query
   // ✅ useState or form library (React Hook Form)
   ```

3. **WebSocket/Real-time data** (use client state + socket library)

4. **File uploads with progress** (use axios with onUploadProgress)

---

## 6. Setup & Configuration

### React Query Setup

```javascript
// App.jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1 * 60 * 1000,     // 1 minute
      cacheTime: 5 * 60 * 1000,     // 5 minutes
      retry: 1,
      refetchOnWindowFocus: false   // Disable focus refetch (optional)
    }
  }
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

---

### SWR Setup

```javascript
import { SWRConfig } from 'swr';

function App() {
  return (
    <SWRConfig value={{
      fetcher: (url) => fetch(url).then(res => res.json()),
      revalidateOnFocus: false,
      shouldRetryOnError: false
    }}>
      <YourApp />
    </SWRConfig>
  );
}
```

---

## 7. Tổng kết Server State

### Key Takeaways

**1. Server state ≠ Client state**
- Don't mix concerns
- Use React Query/SWR for API data
- Use Zustand/Redux for client state

**2. Benefits of React Query/SWR**
- Automatic caching
- Background refetching
- Stale-while-revalidate
- Less boilerplate
- Better UX (instant from cache)

**3. Choose based on needs**
- Simple → SWR (smaller API)
- Complex → React Query (more features, DevTools)

---

### Migration Path

**From manual fetch:**
```
1. Install React Query
2. Replace useState + useEffect with useQuery
3. Remove manual loading/error states
4. Add cache invalidation on mutations
```

**From Redux:**
```
1. Identify server state (API data)
2. Move to React Query
3. Keep client state in Redux
4. 50% less Redux code typically
```

---

**NEXT:** [Redux Guide →](./redux-guide.md)
