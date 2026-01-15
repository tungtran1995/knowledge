# PHẦN 4 — MEMORY MANAGEMENT
Tránh memory leaks và quản lý memory hiệu quả trong JavaScript apps

## 4.1 Memory Management – Tại sao phải quan tâm?

### 📖 JavaScript Memory Model

**JavaScript là garbage-collected language:**
- Developer không cần manual free memory (như C/C++)
- Garbage Collector (GC) tự động reclaim unused memory

**Nhưng vẫn có memory leaks:**
```
Leak = Memory được allocate nhưng không được free
→ Tích lũy theo thời gian
→ App chậm dần
→ Tab crash (Out of Memory)
```

---

### 🎯 Khi nào memory leak là vấn đề?

**Không phải vấn đề:**
- Short-lived pages (landing pages, blogs)
- User không ở lâu trên page

**Là vấn đề nghiêm trọng:**
- Single-Page Apps (SPAs)
- Long-running sessions (Gmail, Figma, dashboards)
- Real-time apps (chat, collaboration tools)

**Symptoms:**
```
After 2 hours usage:
- Page uses 500MB → 2GB RAM
- UI stuttering / janky
- Tab crash
- Browser warning "Page unresponsive"
```

---

### 💡 Mental Model: References & Reachability

**Garbage Collector chỉ free object không "reachable":**

```javascript
// Object được reference → kept in memory
let user = { name: 'John' }; // ✅ Reachable

// Remove reference → GC có thể free
user = null; // ✅ Unreachable → GC reclaims
```

**Problem: Unintended references**
```javascript
const cache = {};

function processUser(user) {
  cache[user.id] = user; // ❌ Reference tồn tại mãi mãi
}

// cache object giữ references đến tất cả users
// → Memory leak
```

---

## 4.2 Common Memory Leak Patterns

### 🔴 Pattern 1: Event Listeners không cleanup

**Problem:**
```javascript
// ❌ Add listener nhưng không remove
function ComponentA() {
  useEffect(() => {
    function handleResize() {
      console.log('Window resized');
    }
    
    window.addEventListener('resize', handleResize);
    // ❌ Thiếu cleanup
  }, []);
}

// Component unmount → listener vẫn tồn tại
// → Component instance không được GC
// → Memory leak
```

**Solution:**
```javascript
// ✅ Always cleanup event listeners
function ComponentA() {
  useEffect(() => {
    function handleResize() {
      console.log('Window resized');
    }
    
    window.addEventListener('resize', handleResize);
    
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);
}
```

---

**Pattern: Multiple listeners**
```javascript
// ✅ Cleanup multiple listeners
function ComponentB() {
  useEffect(() => {
    const handleScroll = () => {};
    const handleKeydown = () => {};
    const handleVisibilityChange = () => {};
    
    window.addEventListener('scroll', handleScroll);
    window.addEventListener('keydown', handleKeydown);
    document.addEventListener('visibilitychange', handleVisibilityChange);
    
    return () => {
      window.removeEventListener('scroll', handleScroll);
      window.removeEventListener('keydown', handleKeydown);
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  }, []);
}
```

---

### 🔴 Pattern 2: Timers không cleanup

**Problem:**
```javascript
// ❌ setInterval không clear
function Clock() {
  const [time, setTime] = useState(new Date());
  
  useEffect(() => {
    setInterval(() => {
      setTime(new Date());
    }, 1000);
    // ❌ Không clearInterval
  }, []);
  
  return <div>{time.toLocaleTimeString()}</div>;
}

// Component unmount → interval vẫn chạy
// → setState trên unmounted component → memory leak
```

**Solution:**
```javascript
// ✅ Clear intervals/timeouts
function Clock() {
  const [time, setTime] = useState(new Date());
  
  useEffect(() => {
    const intervalId = setInterval(() => {
      setTime(new Date());
    }, 1000);
    
    return () => {
      clearInterval(intervalId);
    };
  }, []);
  
  return <div>{time.toLocaleTimeString()}</div>;
}
```

---

**Pattern: Multiple timers**
```javascript
// ✅ Track and clear all timers
function AnimationController() {
  const timerIds = useRef([]);
  
  useEffect(() => {
    // Schedule multiple timers
    timerIds.current.push(setTimeout(() => {}, 1000));
    timerIds.current.push(setTimeout(() => {}, 2000));
    timerIds.current.push(setInterval(() => {}, 500));
    
    return () => {
      // Clear all timers
      timerIds.current.forEach(id => {
        clearTimeout(id);
        clearInterval(id);
      });
      timerIds.current = [];
    };
  }, []);
}
```

---

### 🔴 Pattern 3: Subscriptions không unsubscribe

**Problem: Redux/RxJS subscriptions**
```javascript
// ❌ Subscribe nhưng không unsubscribe
function UserProfile() {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    const subscription = userStore.subscribe((newUser) => {
      setUser(newUser);
    });
    // ❌ Không unsubscribe
  }, []);
}

// Component unmount → subscription vẫn active
// → Callback vẫn chạy → setState on unmounted → leak
```

**Solution:**
```javascript
// ✅ Always unsubscribe
function UserProfile() {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    const subscription = userStore.subscribe((newUser) => {
      setUser(newUser);
    });
    
    return () => {
      subscription.unsubscribe();
    };
  }, []);
}
```

---

**Pattern: RxJS Observable**
```javascript
import { fromEvent } from 'rxjs';
import { debounceTime, map } from 'rxjs/operators';

function SearchInput() {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    const input$ = fromEvent(inputRef.current, 'input').pipe(
      debounceTime(300),
      map(e => e.target.value)
    );
    
    const subscription = input$.subscribe(query => {
      performSearch(query).then(setResults);
    });
    
    return () => {
      subscription.unsubscribe(); // ✅ Critical
    };
  }, []);
}
```

---

### 🔴 Pattern 4: Closures giữ large objects

**Problem:**
```javascript
// ❌ Closure captures large data
function loadUserDashboard(userId) {
  // Load 10MB data
  const userData = await fetchLargeUserData(userId);
  
  // Create event handler
  document.getElementById('refresh').addEventListener('click', () => {
    // Closure captures userData
    console.log('Refreshing for user:', userData.name);
    refresh();
  });
  
  // userData không được GC vì closure reference
}

// 100 page navigations = 1GB trapped in memory
```

**Solution:**
```javascript
// ✅ Extract only needed data
function loadUserDashboard(userId) {
  const userData = await fetchLargeUserData(userId);
  
  // Extract minimal data
  const userName = userData.name; // Only 1 string vs 10MB object
  
  document.getElementById('refresh').addEventListener('click', () => {
    console.log('Refreshing for user:', userName);
    refresh();
  });
  
  // userData can be GC'd now
}
```

---

### 🔴 Pattern 5: Detached DOM nodes

**Problem:**
```javascript
// ❌ Remove DOM node nhưng giữ JS reference
const elements = [];

function createItems() {
  for (let i = 0; i < 1000; i++) {
    const div = document.createElement('div');
    div.innerHTML = `Item ${i}`;
    elements.push(div); // ❌ Store reference
    container.appendChild(div);
  }
}

function clearItems() {
  container.innerHTML = ''; // DOM nodes removed
  // ❌ Nhưng elements array vẫn reference chúng
  // → Detached nodes không được GC
}
```

**Solution:**
```javascript
// ✅ Clear references when removing DOM
const elements = [];

function createItems() {
  for (let i = 0; i < 1000; i++) {
    const div = document.createElement('div');
    div.innerHTML = `Item ${i}`;
    elements.push(div);
    container.appendChild(div);
  }
}

function clearItems() {
  container.innerHTML = '';
  elements.length = 0; // ✅ Clear array
  // Now nodes can be GC'd
}
```

---

## 4.3 React-Specific Memory Leaks

### 🔴 Pattern 1: setState sau khi unmount

**Problem:**
```javascript
// ❌ Async operation sau unmount
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser(userId).then(data => {
      setUser(data); // ❌ Có thể chạy sau unmount
    });
  }, [userId]);
}

// Component unmount trước khi fetch complete
// → setState on unmounted component → warning + leak
```

**Solution 1: Cleanup flag**
```javascript
// ✅ Use cleanup flag
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    let isMounted = true;
    
    fetchUser(userId).then(data => {
      if (isMounted) { // ✅ Check before setState
        setUser(data);
      }
    });
    
    return () => {
      isMounted = false;
    };
  }, [userId]);
}
```

**Solution 2: AbortController**
```javascript
// ✅ Cancel fetch on unmount
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    const abortController = new AbortController();
    
    fetch(`/api/users/${userId}`, {
      signal: abortController.signal
    })
      .then(res => res.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });
    
    return () => {
      abortController.abort(); // ✅ Cancel fetch
    };
  }, [userId]);
}
```

---

### 🔴 Pattern 2: Heavy objects trong state

**Problem:**
```javascript
// ❌ Store processed data in state
function DataVisualization({ rawData }) {
  // rawData = 50MB
  const [processedData, setProcessedData] = useState(null);
  
  useEffect(() => {
    const processed = heavyProcessing(rawData); // 50MB → 100MB
    setProcessedData(processed); // ❌ Duplicate memory
  }, [rawData]);
  
  // Total: 150MB (raw + processed + React internal)
}
```

**Solution:**
```javascript
// ✅ Compute on-demand với useMemo
function DataVisualization({ rawData }) {
  const processedData = useMemo(() => {
    return heavyProcessing(rawData);
  }, [rawData]);
  
  // Only 1 copy in memory (processedData)
  // GC'd when component unmounts
}
```

---

### 🔴 Pattern 3: Global event emitters

**Problem:**
```javascript
// ❌ Global EventEmitter không cleanup
import eventBus from './eventBus';

function NotificationListener() {
  useEffect(() => {
    eventBus.on('notification', handleNotification);
    // ❌ Không off
  }, []);
}

// Mỗi mount → thêm 1 listener
// 10 navigations = 10 listeners cùng chạy
```

**Solution:**
```javascript
// ✅ Cleanup global listeners
import eventBus from './eventBus';

function NotificationListener() {
  useEffect(() => {
    const handleNotification = (data) => {
      console.log('Notification:', data);
    };
    
    eventBus.on('notification', handleNotification);
    
    return () => {
      eventBus.off('notification', handleNotification);
    };
  }, []);
}
```

---

## 4.4 WeakMap & WeakSet – Memory-efficient caching

### 🎯 WeakMap Basics

**Regular Map vs WeakMap:**

```javascript
// ❌ Regular Map prevents GC
const cache = new Map();

function cacheUserData(user) {
  cache.set(user, computeExpensiveData(user));
}

// user object không thể GC vì Map reference
// → Memory leak

// ✅ WeakMap allows GC
const cache = new WeakMap();

function cacheUserData(user) {
  cache.set(user, computeExpensiveData(user));
}

// Khi user object không còn reference khác
// → GC có thể free cả user và cached data ✅
```

**Key differences:**
1. WeakMap keys phải là **objects** (không thể dùng primitives)
2. Keys không enumerable (không iterate được)
3. Keys không prevent GC

---

### 📋 Pattern 1: Component metadata cache

```javascript
// ✅ Cache component metadata
const componentCache = new WeakMap();

function getComponentMetadata(component) {
  if (componentCache.has(component)) {
    return componentCache.get(component);
  }
  
  const metadata = analyzeComponent(component);
  componentCache.set(component, metadata);
  return metadata;
}

// Component unmount → không còn references
// → WeakMap entry tự động removed
```

---

### 📋 Pattern 2: DOM node data storage

```javascript
// ✅ Associate data với DOM nodes
const nodeData = new WeakMap();

function attachDataToNode(node, data) {
  nodeData.set(node, data);
}

function getNodeData(node) {
  return nodeData.get(node);
}

// DOM node removed → data tự động GC'd
// Không cần manual cleanup
```

---

### 📋 Pattern 3: Private properties (legacy pattern)

```javascript
// ✅ Store private data
const privateData = new WeakMap();

class User {
  constructor(name, password) {
    this.name = name;
    
    // Store password privately
    privateData.set(this, { password });
  }
  
  authenticate(inputPassword) {
    const { password } = privateData.get(this);
    return password === inputPassword;
  }
}

// User instance GC'd → private data GC'd
```

---

### 🎯 WeakSet Use Cases

**Track processed objects:**
```javascript
const processedUsers = new WeakSet();

function processUser(user) {
  if (processedUsers.has(user)) {
    return; // Already processed
  }
  
  // ... expensive processing
  
  processedUsers.add(user);
}

// user GC'd → automatically removed from WeakSet
```

---

## 4.5 Debugging Memory Leaks

### 🔍 Chrome DevTools Memory Profiler

**Step 1: Take heap snapshot**
```
1. DevTools → Memory tab
2. Select "Heap snapshot"
3. Click "Take snapshot"
```

**Step 2: Reproduce leak**
```
1. Take snapshot (Baseline)
2. Perform action (e.g., navigate between pages 10x)
3. Force GC (trash icon)
4. Take snapshot (After)
5. Compare snapshots
```

**Step 3: Analyze**
```
Compare view:
- "# New" column → objects created
- "# Deleted" column → objects freed
- "# Delta" column → memory increase

Look for:
- Detached DOM nodes (should be 0 after navigation)
- Arrays/Objects growing
- Event listeners accumulating
```

---

### 📋 Detect detached DOM nodes

**DevTools Console:**
```javascript
// Query detached nodes
queryObjects(HTMLDivElement);

// Check if nodes are detached
// Detached = in memory but not in DOM tree
```

**Heap snapshot filter:**
```
1. Take snapshot
2. Class filter: "Detached"
3. See all detached DOM nodes
```

---

### 📋 Allocation Timeline

**Track memory over time:**
```
1. DevTools → Memory → Allocation instrumentation on timeline
2. Start recording
3. Perform user actions
4. Stop recording
5. Analyze blue bars (allocations that weren't freed)
```

**Interpret:**
- Blue bars = allocations
- Gray bars = freed
- Blue bars staying = potential leaks

---

## 4.6 Memory Optimization Patterns

### ✅ Pattern 1: Object pooling

**Reuse objects thay vì create/destroy:**
```javascript
// ❌ Create nhiều objects
function animate() {
  for (let i = 0; i < 1000; i++) {
    const particle = {
      x: Math.random() * 800,
      y: Math.random() * 600,
      vx: Math.random() * 2,
      vy: Math.random() * 2
    };
    particles.push(particle);
  }
}

// GC pressure cao

// ✅ Object pool
class ParticlePool {
  constructor(size) {
    this.pool = Array(size).fill(null).map(() => ({
      x: 0, y: 0, vx: 0, vy: 0, active: false
    }));
  }
  
  get() {
    return this.pool.find(p => !p.active);
  }
  
  release(particle) {
    particle.active = false;
  }
}

const pool = new ParticlePool(1000);

function animate() {
  for (let i = 0; i < 1000; i++) {
    const particle = pool.get();
    if (particle) {
      particle.x = Math.random() * 800;
      particle.y = Math.random() * 600;
      particle.active = true;
    }
  }
}

// Reuse objects → less GC
```

---

### ✅ Pattern 2: Pagination thay vì load all

```javascript
// ❌ Load 10000 items
function loadAllUsers() {
  const users = await fetch('/api/users?limit=10000');
  setUsers(users); // 50MB in memory
}

// ✅ Pagination
function loadUsers(page = 1, pageSize = 50) {
  const users = await fetch(`/api/users?page=${page}&limit=${pageSize}`);
  setUsers(users); // Only ~250KB
}
```

---

### ✅ Pattern 3: Clear large data structures

```javascript
// ✅ Clear arrays/maps khi không dùng
function processLargeDataset(data) {
  const intermediate = new Map();
  
  // ... processing
  
  // Clear before function exits
  intermediate.clear();
  intermediate = null;
}
```

---

### ✅ Pattern 4: Debounce expensive computations

```javascript
// ❌ Compute on every change
function SearchResults({ query }) {
  const results = expensiveSearch(query); // Re-run mỗi keystroke
}

// ✅ Debounce
function SearchResults({ query }) {
  const [debouncedQuery] = useDebounce(query, 300);
  const results = useMemo(() => 
    expensiveSearch(debouncedQuery), 
    [debouncedQuery]
  );
}

// Reduce memory churn
```

---

## 🎓 Tổng kết Memory Management

### ✅ Cleanup Checklist

**Mỗi khi add listener/subscription/timer:**
```javascript
useEffect(() => {
  // ✅ Setup
  const listener = ...;
  const timer = ...;
  const subscription = ...;
  
  // ✅ ALWAYS cleanup
  return () => {
    removeListener(listener);
    clearTimeout(timer);
    subscription.unsubscribe();
  };
}, []);
```

---

### 📊 Common Memory Leak Sources (Priority)

| Source | Frequency | Impact | Fix difficulty |
|--------|-----------|--------|----------------|
| **Event listeners** | Very High | High | Easy |
| **Timers (setInterval)** | High | Medium | Easy |
| **Subscriptions** | High | Medium | Easy |
| **setState after unmount** | High | Low | Easy |
| **Closures** | Medium | High | Medium |
| **Detached DOM** | Medium | Medium | Medium |
| **Global caches** | Low | High | Hard |

---

### 🎯 Decision Framework

```
Memory usage tăng theo thời gian?
├─ DevTools Memory → Heap snapshot → Compare
│  ├─ Detached DOM? → Check DOM cleanup
│  ├─ Event listeners? → Check cleanup
│  └─ Large arrays/objects? → Check state management
│
Long-running SPA?
├─ ✅ Setup cleanup cho:
│  ├─ Event listeners
│  ├─ Timers
│  ├─ Subscriptions
│  └─ Async operations
│
Heavy caching?
├─ Consider WeakMap/WeakSet
└─ Add expiration logic
```

---

### 🔧 Tools & Resources

**Chrome DevTools:**
- Memory Profiler (heap snapshots)
- Performance Monitor (real-time memory)
- Allocation Timeline

**Libraries:**
- `why-did-you-render` - Detect unnecessary re-renders
- `@welldone-software/why-did-you-render` - React specific

**Best practices:**
1. ✅ Test long sessions (2+ hours)
2. ✅ Monitor memory in production (RUM)
3. ✅ Heap snapshot before/after major features
4. ✅ Review cleanup code in PRs
5. ✅ Use lint rules (eslint-plugin-react-hooks)

---

**DOCUMENT COMPLETE ✅**

*Version: 1.0*  
*Last updated: 2026-01-02*  
*Focus: MEMORY MANAGEMENT*  
*Related: Measure Performance, Network Optimization, Runtime Performance*
