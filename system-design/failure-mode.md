# Failure Modes & Recovery Strategies

> **Core idea:** Everything fails eventually. Plan for failure, not just success.

---

## 1. What is a Failure Mode?

### Definition

```
Failure Mode = A specific way something can go wrong

Example:
- API timeout
- Network disconnect
- Cache miss
- State update race condition
```

**NOT just bugs:** Expected failures that happen in production.

---

### Why Plan for Failures?

```
Optimistic mindset: "User has fast network, API always works"
→ White screen when something fails

Defensive mindset: "What if API times out?"
→ Show cached data + retry button
```

**Goal:** **Graceful degradation** instead of catastrophic failure.

---

## 2. Common Frontend Failure Modes

### Failure 1: API Request Timeout

**What fails:**
```
fetch('/api/data')
→ Network slow / server overloaded
→ Request hangs forever
```

**Impact:** Infinite loading spinner, user stuck.

---

**Recovery strategies:**

**Option A: Timeout + Retry**
```javascript
async function fetchWithTimeout(url, timeout = 5000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);
  
  try {
    const response = await fetch(url, { signal: controller.signal });
    clearTimeout(timeoutId);
    return response;
  } catch (error) {
    if (error.name === 'AbortError') {
      // Timeout occurred
      throw new Error('Request timeout');
    }
    throw error;
  }
}

// Usage with retry
const { data, error } = useQuery('posts', fetchPosts, {
  timeout: 5000,
  retry: 3,
  retryDelay: 1000
});
```

---

**Option B: Show Cached Data**
```javascript
const { data, error, isLoading } = useQuery('posts', fetchPosts);

if (error) {
  return (
    <div>
      <CachedPosts /> {/* Show stale data */}
      <Banner>Network error. Showing cached data. <Retry /></Banner>
    </div>
  );
}
```

---

### Failure 2: WebSocket Disconnect

**What fails:**
```
WebSocket connection
→ User switches network (WiFi → 4G)
→ Connection drops
```

**Impact:** Real-time updates stop, user sees stale data.

---

**Recovery strategies:**

**Auto-reconnect with exponential backoff**
```javascript
class WebSocketManager {
  constructor(url) {
    this.url = url;
    this.reconnectDelay = 1000; // Start with 1s
    this.maxDelay = 30000; // Max 30s
    this.connect();
  }
  
  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onclose = () => {
      console.log(`Reconnecting in ${this.reconnectDelay}ms...`);
      
      setTimeout(() => {
        this.reconnectDelay = Math.min(
          this.reconnectDelay * 2,
          this.maxDelay
        );
        this.connect();
      }, this.reconnectDelay);
    };
    
    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectDelay = 1000; // Reset on success
    };
  }
}
```

**UI feedback:**
```javascript
function ChatApp() {
  const [connected, setConnected] = useState(true);
  
  return (
    <>
      {!connected && (
        <Banner>Reconnecting... <Spinner /></Banner>
      )}
      <ChatMessages />
    </>
  );
}
```

---

### Failure 3: Cache Stampede

**What fails:**
```
1000 users hit page simultaneously
Cache expired for key "popular-posts"
All 1000 requests hit API at once
→ Server overload
```

**Impact:** API crashes, all users see errors.

---

**Recovery strategies:**

**Request deduplication**
```javascript
// React Query built-in
const { data } = useQuery('posts', fetchPosts);

// If 1000 components mount simultaneously:
// → Only 1 API call (others wait for result)
```

**Stale-while-revalidate**
```javascript
const { data } = useQuery('posts', fetchPosts, {
  staleTime: 5 * 60 * 1000, // Fresh for 5min
  cacheTime: 30 * 60 * 1000 // Keep in cache 30min
});

// If cache expired:
// → Return stale data immediately
// → Fetch fresh in background
// → Update when ready
```

---

### Failure 4: Race Condition in State Updates

**What fails:**
```
User clicks "Like" → API call starts
User clicks "Unlike" before first call returns
First call returns → Sets liked = true
Second call returns → Sets liked = false

Result: UI shows "Unlike" but server has "Like"
```

**Impact:** UI out of sync with server.

---

**Recovery strategies:**

**Option A: Disable during pending**
```javascript
function LikeButton({ postId }) {
  const [isPending, startTransition] = useTransition();
  
  const mutation = useMutation(toggleLike);
  
  const handleClick = () => {
    startTransition(() => {
      mutation.mutate(postId);
    });
  };
  
  return (
    <button 
      onClick={handleClick} 
      disabled={isPending || mutation.isLoading}
    >
      {isPending ? 'Updating...' : 'Like'}
    </button>
  );
}
```

---

**Option B: Cancel previous request**
```javascript
const mutation = useMutation(toggleLike, {
  onMutate: async (postId) => {
    // Cancel any outgoing queries
    await queryClient.cancelQueries(['post', postId]);
    
    // Optimistic update
    const previous = queryClient.getQueryData(['post', postId]);
    queryClient.setQueryData(['post', postId], (old) => ({
      ...old,
      liked: !old.liked
    }));
    
    return { previous };
  },
  
  onError: (err, postId, context) => {
    // Rollback on error
    queryClient.setQueryData(['post', postId], context.previous);
  }
});
```

---

### Failure 5: Memory Leak (Infinite Scroll)

**What fails:**
```
User scrolls down → Load more items
User scrolls down → Load more items
... after 1000 items ...
→ Browser crashes (too much memory)
```

**Impact:** Tab freezes, user loses work.

---

**Recovery strategies:**

**Virtual scrolling**
```javascript
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={10000} // Can handle huge lists
  itemSize={50}
>
  {({ index, style }) => (
    <div style={style}>
      <Item data={items[index]} />
    </div>
  )}
</FixedSizeList>

// Only renders ~20 visible items
// Old items unmounted automatically
```

---

**Pagination fallback**
```javascript
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery(
  'posts',
  fetchPosts,
  {
    getNextPageParam: (lastPage) => lastPage.nextCursor
  }
);

// If total items > 1000:
// → Switch to pagination
if (data.pages.length > 10) {
  return <PaginationView data={data} />;
}
```

---

### Failure 6: Form Submission During Navigation

**What fails:**
```
User fills form
User clicks "Submit"
User clicks "Back" before API returns
→ Navigation happens
→ API returns (but component unmounted)
→ setState on unmounted component warning
```

**Impact:** Memory leak, potential data loss.

---

**Recovery strategies:**

**Cleanup on unmount**
```javascript
function FormComponent() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function submit() {
      const result = await api.submitForm(data);
      
      if (!cancelled) {
        setData(result); // Only update if still mounted
      }
    }
    
    submit();
    
    return () => {
      cancelled = true; // Cleanup
    };
  }, []);
}
```

---

**AbortController**
```javascript
function FormComponent() {
  const mutation = useMutation(submitForm);
  
  useEffect(() => {
    return () => {
      // Cancel pending request on unmount
      mutation.reset();
    };
  }, []);
}
```

---

## 3. Failure Mode Checklist

### By Component Type

**Data Fetching:**
- ☐ Timeout (show retry button)
- ☐ Network error (show cached data)
- ☐ 500 error (show error boundary)
- ☐ Empty response (show empty state)

**Real-time (WebSocket):**
- ☐ Disconnect (auto-reconnect with backoff)
- ☐ Message ordering (sequence numbers)
- ☐ Duplicate messages (deduplication)

**Forms:**
- ☐ Validation error (inline feedback)
- ☐ Submission during navigation (cleanup)
- ☐ Double submit (disable button)
- ☐ Network error during submit (retry or save draft)

**Lists:**
- ☐ Memory leak (virtual scrolling)
- ☐ Infinite scroll failure (pagination fallback)
- ☐ Slow rendering (debounce filter)

---

# Error Boundaries - Bổ sung vào `failure-modes.md`

---

## Thêm vào `failure-modes.md` (sau section "Common Frontend Failure Modes")

---

## 8. Error Boundaries (Component Crash Handling)

### The Problem: Component Crashes = White Screen

```javascript
// Component throws error
function BrokenWidget() {
  const data = null;
  return <div>{data.name}</div>; // TypeError: Cannot read property 'name' of null
}

// Without Error Boundary:
// → Entire app crashes
// → White screen
// → User loses all work
```

---

### The Solution: Error Boundaries

```javascript
// Catch errors, show fallback
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };
  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error, errorInfo) {
    // Log to error tracking service
    console.error('Error caught:', error, errorInfo);
    // Send to Sentry, LogRocket, etc.
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

// Usage: Wrap risky components
<ErrorBoundary>
  <BrokenWidget />
</ErrorBoundary>

// Result: Widget crashes → Fallback shown → Rest of app works
```

---

### Placement Strategy

**Strategy 1: Route-level (Coarse)**

```javascript
function App() {
  return (
    <Routes>
      <Route path="/dashboard" element={
        <ErrorBoundary fallback={<DashboardError />}>
          <Dashboard />
        </ErrorBoundary>
      } />
      
      <Route path="/settings" element={
        <ErrorBoundary fallback={<SettingsError />}>
          <Settings />
        </ErrorBoundary>
      } />
    </Routes>
  );
}

// Benefit: Page crashes, other pages still work
// Trade-off: Entire page shows error (not granular)
```

---

**Strategy 2: Component-level (Fine-grained)**

```javascript
function Dashboard() {
  return (
    <div>
      <Header /> {/* No boundary - critical */}
      
      <ErrorBoundary fallback={<WidgetError />}>
        <ComplexWidget /> {/* Widget crashes, dashboard still works */}
      </ErrorBoundary>
      
      <ErrorBoundary fallback={<ChartError />}>
        <ExpensiveChart /> {/* Chart crashes, widget still works */}
      </ErrorBoundary>
      
      <Footer /> {/* No boundary - critical */}
    </div>
  );
}

// Benefit: Granular (widget crashes, rest works)
// Trade-off: More boundaries (verbosity)
```

---

**Strategy 3: Top-level (Last Resort)**

```javascript
function App() {
  return (
    <ErrorBoundary fallback={<AppCrashed />}>
      <Routes />
    </ErrorBoundary>
  );
}

// Catches anything not caught by lower boundaries
// Last line of defense before white screen
```

---

### Decision Matrix

| Placement | When to Use | Benefit | Trade-off |
|-----------|-------------|---------|-----------|
| **Top-level** | Always | Prevent white screen | Coarse (entire app error) |
| **Route-level** | Multi-page app | Page isolation | Entire page error |
| **Component-level** | Complex widgets | Fine-grained | Verbose |

**Recommendation:** Use all three (defense in depth)

```
Top-level (catch all)
  → Route-level (page isolation)
    → Component-level (widget isolation)
```

---

### What Error Boundaries DON'T Catch

```javascript
// ❌ Error boundaries do NOT catch:

// 1. Event handlers
<button onClick={() => {
  throw new Error('Click error'); // NOT caught
}}>
  Click
</button>

// Fix: Use try/catch
<button onClick={() => {
  try {
    riskyOperation();
  } catch (error) {
    handleError(error);
  }
}}>
  Click
</button>
```

---

```javascript
// 2. Async code
useEffect(() => {
  async function fetchData() {
    throw new Error('Async error'); // NOT caught
  }
  fetchData();
}, []);

// Fix: Use error state
useEffect(() => {
  async function fetchData() {
    try {
      const data = await api.fetch();
    } catch (error) {
      setError(error); // Handle in state
    }
  }
  fetchData();
}, []);
```

---

```javascript
// 3. Server-side rendering (SSR)
// Error during SSR not caught by client Error Boundary

// Fix: Server-side error handling
app.use((err, req, res, next) => {
  res.status(500).render('error', { error: err });
});
```

---

```javascript
// 4. Errors in Error Boundary itself
class ErrorBoundary extends React.Component {
  componentDidCatch(error) {
    throw new Error('Boundary error'); // NOT caught (infinite loop risk)
  }
}

// Fix: Don't throw in Error Boundary
```

---

### Advanced Patterns

**Pattern 1: Retry Mechanism**

```javascript
class ErrorBoundary extends React.Component {
  state = { hasError: false, errorCount: 0 };
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  componentDidCatch() {
    this.setState(prev => ({ errorCount: prev.errorCount + 1 }));
  }
  
  resetError = () => {
    this.setState({ hasError: false });
  };
  
  render() {
    const { hasError, errorCount } = this.state;
    
    if (hasError) {
      if (errorCount > 3) {
        // Too many errors, give up
        return <FatalError />;
      }
      
      return (
        <div>
          <p>Error occurred (attempt {errorCount}/3)</p>
          <button onClick={this.resetError}>Retry</button>
        </div>
      );
    }
    
    return this.props.children;
  }
}
```

---

**Pattern 2: Error Telemetry**

```javascript
import * as Sentry from '@sentry/react';

class ErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // Send to error tracking
    Sentry.captureException(error, {
      contexts: {
        react: {
          componentStack: errorInfo.componentStack
        }
      },
      tags: {
        boundary: this.props.name // Which boundary caught it
      }
    });
  }
  
  render() { /* ... */ }
}

// Usage with name
<ErrorBoundary name="dashboard-widget">
  <Widget />
</ErrorBoundary>

// Sentry dashboard shows: "Error in dashboard-widget boundary"
```

---

**Pattern 3: Contextual Fallbacks**

```javascript
function ErrorFallback({ error, resetError, context }) {
  const messages = {
    widget: 'This widget encountered an error',
    chart: 'Chart failed to load',
    form: 'Form submission failed'
  };
  
  return (
    <div>
      <h3>{messages[context] || 'Something went wrong'}</h3>
      <details>
        <summary>Error details</summary>
        <pre>{error.message}</pre>
      </details>
      <button onClick={resetError}>Try again</button>
    </div>
  );
}

// Usage
<ErrorBoundary 
  fallback={(error, reset) => (
    <ErrorFallback error={error} resetError={reset} context="widget" />
  )}
>
  <Widget />
</ErrorBoundary>
```

---

### React 19: Error Boundary Hook (Future)

```javascript
// React 19+ (experimental)
import { useErrorBoundary } from 'react';

function MyComponent() {
  const [error, resetError] = useErrorBoundary();
  
  if (error) {
    return (
      <div>
        <p>Error: {error.message}</p>
        <button onClick={resetError}>Reset</button>
      </div>
    );
  }
  
  return <div>Normal content</div>;
}

// Simpler than class components
// Still experimental (not stable yet)
```

---

### Error Boundary Checklist

**Setup:**
- ☐ Top-level boundary (catch all)
- ☐ Route-level boundaries (page isolation)
- ☐ Component-level for risky widgets (optional)
- ☐ Error telemetry integrated (Sentry, LogRocket)

**Fallback UI:**
- ☐ User-friendly message (not technical jargon)
- ☐ Retry button (if error might be temporary)
- ☐ Contact support link (if persistent)
- ☐ Show error details in dev mode only

**What NOT to wrap:**
- ☐ Event handlers (use try/catch)
- ☐ Async code (use error state)
- ☐ Critical UI (header, nav - should never crash)

---

### Real-World Example

```javascript
// Production setup
import { ErrorBoundary as SentryBoundary } from '@sentry/react';

function App() {
  return (
    // Top-level: Sentry integration
    <SentryBoundary 
      fallback={<AppCrashed />}
      showDialog // Show Sentry feedback dialog
    >
      <Routes>
        {/* Route-level: Custom fallbacks */}
        <Route path="/dashboard" element={
          <ErrorBoundary fallback={<DashboardError />}>
            <Dashboard />
          </ErrorBoundary>
        } />
      </Routes>
    </SentryBoundary>
  );
}

function Dashboard() {
  return (
    <div>
      <Header />
      
      {/* Component-level: Granular */}
      <ErrorBoundary 
        fallback={<WidgetError />}
        onError={(error) => {
          // Custom logging
          analytics.track('widget_error', { error: error.message });
        }}
      >
        <ExpensiveWidget />
      </ErrorBoundary>
      
      <Footer />
    </div>
  );
}
```

---

### Mental Model: Blast Radius

```
Without boundaries:
Component error → App crash → White screen → User loses work

With boundaries:
Component error → Boundary catches → Fallback shown → Rest works

Layered boundaries:
Widget error → Component boundary catches → Widget fallback
              → If boundary fails → Route boundary catches → Page fallback
                                   → If that fails → App boundary → App fallback
                                                    → Last resort: White screen (rare)

Goal: Minimize blast radius
```

---

### Key Insight

> **Error Boundaries are like try/catch for React render tree**

Use them like you use try/catch in regular code: wrap risky operations.

**Action:** Add top-level Error Boundary to every app. Then add route/component boundaries for critical widgets.

---


## 4. General Recovery Patterns

### Pattern 1: Error Boundaries

```javascript
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  componentDidCatch(error, errorInfo) {
    logErrorToService(error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return <ErrorFallback />;
    }
    return this.props.children;
  }
}

// Usage: Wrap risky components
<ErrorBoundary>
  <ComplexWidget />
</ErrorBoundary>
```

---

### Pattern 2: Retry with Exponential Backoff

```javascript
async function fetchWithRetry(fn, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === retries - 1) throw error;
      
      const delay = Math.pow(2, i) * 1000; // 1s, 2s, 4s
      await sleep(delay);
    }
  }
}
```

---

### Pattern 3: Fallback Data

```javascript
const { data, error } = useQuery('critical-data', fetchData);

// Always have fallback
const displayData = data ?? fallbackData ?? defaultData;

return <View data={displayData} />;
```

---

### Pattern 4: Optimistic Updates

```javascript
// Assume success, rollback if fails
const mutation = useMutation(updatePost, {
  onMutate: async (newData) => {
    const previous = queryClient.getQueryData('post');
    queryClient.setQueryData('post', newData); // Optimistic
    return { previous };
  },
  
  onError: (err, newData, context) => {
    queryClient.setQueryData('post', context.previous); // Rollback
  }
});
```

---

## 5. Mental Models

### Model 1: Blast Radius

```
Failure impact:
- Component crash → Error boundary catches → Page still works
- Page crash → Top-level boundary → App still works
- App crash → White screen (worst case)

Strategy: Contain failures at lowest level
```

---

### Model 2: Graceful Degradation

```
Ideal:     Real-time updates via WebSocket
Degraded:  Polling every 30s
Fallback:  Manual refresh button
Worst:     Cached data only

Always have fallback → Never total failure
```

---

### Model 3: Fail Fast vs Retry

```
Fail fast: User input validation (instant feedback)
Retry:     Network requests (temporary failures)

Rule: If likely temporary → Retry
      If permanent error → Fail fast
```

---

## 6. Testing Failure Modes

### Simulate Failures

**Network timeout:**
```javascript
// Chrome DevTools → Network → Throttling → Offline
// Or
await fetch('/api', { signal: AbortSignal.timeout(100) });
```

**Slow API:**
```javascript
// Mock API with delay
msw.get('/api/posts', async (req, res, ctx) => {
  await delay(5000); // Simulate slow response
  return res(ctx.json(posts));
});
```

**Race condition:**
```javascript
// Click button rapidly
await userEvent.click(likeButton);
await userEvent.click(likeButton);
await userEvent.click(likeButton);
// Does UI stay consistent?
```

---

## 7. Tổng Kết

### Core Principles

1. **Expect failures:** Network, API, user actions all fail
2. **Plan recovery:** Timeout, retry, fallback, error boundaries
3. **User feedback:** Show what's happening (loading, error, retrying)
4. **Graceful degradation:** Cached data > Error message > White screen

---

### Quick Reference

| Failure | Recovery |
|---------|----------|
| API timeout | Timeout + retry + cached data |
| WebSocket disconnect | Auto-reconnect + exponential backoff |
| Cache stampede | Request deduplication + SWR |
| Race condition | Disable button or cancel previous |
| Memory leak | Virtual scrolling or pagination |
| Component crash | Error boundary |

---

