# Caching Strategies for Frontend

> **Goal:** Minimize network requests, maximize perceived performance.

---

## 1. Caching Layers

### The Full Stack

```
1. Browser Memory Cache (React Query, SWR)
   ↓ miss
2. Service Worker Cache (offline capability)
   ↓ miss
3. Browser HTTP Cache (static assets)
   ↓ miss
4. CDN Edge Cache (global distribution)
   ↓ miss
5. API Server
```

**Each layer = cache hit = faster response.**

---

## 2. Strategies

### Cache-First (Offline-First)

```
1. Check cache
2. If hit → Return immediately
3. If miss → Fetch from network → Update cache
```

**Use when:** Data rarely changes, offline support critical.

**Example:** News articles, documentation.

---

### Network-First

```
1. Try network
2. If success → Update cache → Return
3. If fail → Return stale cache
```

**Use when:** Fresh data important, but offline fallback needed.

**Example:** User profile, account balance.

---

### Stale-While-Revalidate (SWR)

```
1. Return stale cache immediately (instant UI)
2. Fetch fresh data in background
3. Update cache when fresh data arrives
```

**Use when:** Best UX, data can be slightly stale.

**Example:** Social media feeds, dashboards.

**React Query default strategy!**

---

## 3. Cache Invalidation

### The Hard Problem

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

---

### Strategies

**Time-based (TTL):**
```javascript
const { data } = useQuery('posts', fetchPosts, {
  staleTime: 5 * 60 * 1000 // Fresh for 5 minutes
});
```

**Event-based:**
```javascript
// Invalidate on mutation
const mutation = useMutation(createPost, {
  onSuccess: () => {
    queryClient.invalidateQueries('posts');
  }
});
```

**Manual:**
```javascript
// Force refresh
queryClient.invalidateQueries('posts');
```

---

## 4. Service Worker Caching

### Workbox Strategies

```javascript
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';

// Images: Cache-first
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({ cacheName: 'images' })
);

// API: Network-first
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api' })
);

// HTML: Stale-while-revalidate
registerRoute(
  ({ request }) => request.destination === 'document',
  new StaleWhileRevalidate({ cacheName: 'pages' })
);
```

---

## 5. Best Practices

### Practice 1: Cache Key Design

```javascript
// ❌ Bad (too generic)
useQuery('user', fetchUser);

// ✅ Good (specific)
useQuery(['user', userId], () => fetchUser(userId));

// ✅ Better (with dependencies)
useQuery(
  ['user', userId, { include: 'posts' }],
  () => fetchUser(userId, { include: 'posts' })
);
```

---

### Practice 2: Optimistic Updates

```javascript
const mutation = useMutation(updatePost, {
  onMutate: async (newPost) => {
    // Cancel outgoing queries
    await queryClient.cancelQueries(['post', newPost.id]);
    
    // Save current state
    const previous = queryClient.getQueryData(['post', newPost.id]);
    
    // Optimistically update
    queryClient.setQueryData(['post', newPost.id], newPost);
    
    return { previous };
  },
  
  onError: (err, newPost, context) => {
    // Rollback on error
    queryClient.setQueryData(['post', newPost.id], context.previous);
  },
  
  onSettled: (newPost) => {
    // Refetch to sync with server
    queryClient.invalidateQueries(['post', newPost.id]);
  }
});
```

---

## Tổng Kết

**Caching = Performance multiplier**

**Choose strategy:**
- Offline-first app → Cache-first
- Real-time data → Network-first
- Best UX → Stale-while-revalidate

**Next:** [Scalability Patterns →](./scalability.md)
