# Memory Management - Quản lý bộ nhớ trong JavaScript

> **Mục tiêu**: Hiểu sâu về cơ chế quản lý bộ nhớ, garbage collection, và tối ưu hóa performance trong production.

---

## 📚 Table of Contents

1. [Memory Model & Lifecycle](#1-memory-model--lifecycle)
2. [Garbage Collection - Cơ Chế Hoạt Động](#2-garbage-collection---cơ-chế-hoạt-động)
3. [Memory Leaks - Phân Tích & Phòng Tránh](#3-memory-leaks---phân-tích--phòng-tránh)
4. [WeakMap & WeakSet - Advanced Use Cases](#4-weakmap--weakset---advanced-use-cases)
5. [Chrome DevTools - Profiling Techniques](#5-chrome-devtools---profiling-techniques)
6. [Production Optimization Strategies](#6-production-optimization-strategies)

---

## 1. Memory Model & Lifecycle

### 1.1. JavaScript Memory Architecture

**Hai vùng nhớ chính trong JavaScript Runtime**:

#### Stack Memory (Call Stack)
- **Lưu trữ**: Primitive values (number, string, boolean, null, undefined, symbol, bigint) và object references
- **Kích thước**: Nhỏ, cố định (~1MB trên most platforms)
- **Cơ chế quản lý**: Automatic LIFO (Last In First Out) - tự động push/pop khi function call/return
- **Tốc độ truy cập**: Rất nhanh (CPU cache-friendly, contiguous memory)
- **Scope**: Function-level, tự động cleanup khi function returns
- **Allocation**: Compile-time determined size

**Stack Overflow xảy ra khi**:
- Recursive calls quá sâu (typically ~10,000-15,000 calls)
- Too many function parameters/local variables

#### Heap Memory (Managed Heap)
- **Lưu trữ**: Objects, arrays, functions, closures - mọi reference types
- **Kích thước**: Lớn, dynamic allocation
  - **Browsers**: ~1-4GB (varies by device RAM)
  - **Node.js**: Default ~1.4GB (32-bit), ~4GB (64-bit)
  - **Node.js với flags**: Unlimited (--max-old-space-size)
- **Cơ chế quản lý**: Garbage Collection (Mark-Sweep-Compact)
- **Tốc độ truy cập**: Chậm hơn Stack (non-contiguous, pointer indirection)
- **Scope**: Determined by reachability, không phải scope
- **Allocation**: Runtime, dynamic sizing

**Heap exhaustion symptoms**:
- "JavaScript heap out of memory" error
- Process.memoryUsage().heapUsed > process.memoryUsage().heapTotal * 0.9

### 1.2. Memory Lifecycle Phases

#### Phase 1: Allocation
```
Khi nào: Tạo mới objects, arrays, functions
Cơ chế: 
- Stack: Tự động khi function call
- Heap: Garbage collector allocates
```

#### Phase 2: Usage
```
Đọc/ghi properties
Function calls
Array operations
```

#### Phase 3: Release
```
Automatic: Garbage collector thu hồi unreachable objects
Manual: Không có (không như C/C++)
```

### 1.3. Hidden Classes (V8 Shape Optimization)

**Khái niệm**: V8 tạo internal "hidden classes" (hay "shapes"/"maps") để optimize property access performance

**Cơ chế hoạt động**:
```javascript
// Cùng hidden class
function Point(x, y) {
  this.x = x;  // Hidden class C1: {} -> {x}
  this.y = y;  // Hidden class C2: {x} -> {x, y}
}

const p1 = new Point(1, 2);  // Uses C2
const p2 = new Point(3, 4);  // Reuses C2 (cùng shape!)
// → Inline caching works perfectly

// Khác hidden class
const p3 = {};
p3.y = 5;  // Hidden class C3: {} -> {y}
p3.x = 6;  // Hidden class C4: {y} -> {y, x}  (khác thứ tự!)
// → Different shape → inline cache miss
```

**Best Practices** (Critical for Performance):
1. **Khởi tạo tất cả properties trong constructor**
   ```javascript
   // ✅ Good: Predictable shape
   class User {
     constructor(name, age) {
       this.name = name;
       this.age = age;
       this.score = 0;  // Initialize even if 0/null
     }
   }
   
   // ❌ Bad: Unpredictable shape
   class User {
     constructor(name) {
       this.name = name;
       // age và score added later → multiple shapes
     }
   }
   ```

2. **Thêm properties theo cùng thứ tự**
   ```javascript
   // ✅ Good: Same order
   const obj1 = {}; obj1.x = 1; obj1.y = 2;
   const obj2 = {}; obj2.x = 3; obj2.y = 4;
   // → Cùng transition chain C0 -> C1(x) -> C2(x,y)
   
   // ❌ Bad: Different order  
   const obj3 = {}; obj3.y = 5; obj3.x = 6;
   // → Different chain C0 -> C3(y) -> C4(y,x)
   ```

3. **Tránh `delete` properties** (causes deoptimization)
   ```javascript
   // ❌ Bad: Delete creates "sparse" property map
   delete obj.property;
   
   // ✅ Good: Set to null/undefined
   obj.property = null;
   ```

4. **Tránh thay đổi property types**
   ```javascript
   // ❌ Bad: Type changes
   obj.value = 42;      // Number
   obj.value = "text";  // String → deoptimization
   
   // ✅ Good: Consistent types
   obj.numberValue = 42;
   obj.stringValue = "text";
   ```

**Performance Impact**:
- **Same hidden class** → Inline caching → **10-100x faster** property access
- **Different hidden classes** → Megamorphic → Deoptimization → Dictionary mode
- **Measurement**: Use `%DebugPrint(obj)` (Node.js --allow-natives-syntax) để xem hidden class

### 1.4. Inline Caching - V8 Fast Property Access

**Cơ chế**: V8 caches kết quả của property lookups tại call site

**IC States (Inline Cache States)**:

1. **Monomorphic** (Best - Single Shape):
   ```javascript
   function getX(obj) { return obj.x; }
   
   // All calls với cùng hidden class
   getX({x: 1, y: 2});  // Cache: "getX always accesses Point.x at offset 0"
   getX({x: 3, y: 4});  // Hit cache → Rất nhanh!
   ```
   **Performance**: ~1-2 CPU cycles

2. **Polymorphic** (Good - 2-4 Shapes):
   ```javascript
   getX({x: 1, y: 2});      // Shape A
   getX({x: 3, z: 4});      // Shape B  
   getX({x: 5, w: 6});      // Shape C
   // Cache: "Check if shape A, B, or C, then access accordingly"
   ```
   **Performance**: ~4-8 CPU cycles (multiple checks)
   **Still acceptable**: V8 handles this well

3. **Megamorphic** (Bad - >4 Shapes):
   ```javascript
   // 5+ different shapes
   getX({x: 1, a: 2});
   getX({x: 3, b: 4});
   getX({x: 5, c: 6});
   getX({x: 7, d: 8});
   getX({x: 9, e: 10});  // 5th shape → Megamorphic!
   // Cache: "Give up, do full property lookup every time"
   ```
   **Performance**: ~100+ CPU cycles (dictionary lookup)
   **Result**: **10-100x slower** than monomorphic!

**Practical Guidelines**:
- **Target**: Monomorphic (1 shape) or Polymorphic (2-4 shapes)
- **Avoid**: Megamorphic (>4 shapes)
- **Detection**: Use `--trace-ic` flag (Node.js) để track IC states
- **Rule**: Keep objects with same structure together in hot paths

---

## 2. Garbage Collection - Cơ Chế Hoạt Động

### 2.1. Evolution of GC Algorithms

#### Era 1: Reference Counting (Obsolete)
```
Mechanism: Count references to each object
Problem: Circular references leak memory
Status: Không còn dùng
```

#### Era 2: Mark-and-Sweep (Foundation)
```
Phase 1 - Mark: Traverse from roots, mark reachable
Phase 2 - Sweep: Reclaim unmarked objects
Pros: Handles circular refs
Cons: Stop-the-world pause
```

#### Era 3: Generational GC (Modern - V8)
```
Observation: Most objects die young
Strategy: Separate young/old generations
Result: Faster, less pause time
```

### 2.2. V8 Generational GC Architecture

**Weak Generational Hypothesis**: "Most objects die young" - 90-95% objects tồn tại < 1 GC cycle

#### Young Generation (New Space / Nursery)
```
Size: 1-16 MB (semi-space design: 2 equal halves)
Algorithm: Scavenge (Cheney's copy collector)
Frequency: Rất thường xuyên (mỗi vài milliseconds)
Pause time: ~1-5ms (rất ngắn!)
Objects: Mới tạo, tuổi thọ ngắn
Survival mechanism: Survive 2 GC cycles → promoted to Old Gen
Throughput: ~10-20 MB/ms (very fast)
```

**Scavenge Algorithm** (Copy Collection):
```
1. Young Gen chia làm 2 semi-spaces: FROM-space và TO-space
2. Allocations vào FROM-space
3. Khi FROM-space full:
   - Copy live objects sang TO-space
   - Swap FROM ⇔ TO
   - FROM-space (old TO) becomes free
4. Dead objects tự động reclaimed (không copy)
```

**Why so fast?**: Chỉ copy live objects (~5-10% typically), ignore dead majority

#### Old Generation (Old Space / Tenured)
```
Size: Large (up to several GB, đa số heap)
Algorithm: Mark-Sweep-Compact (tri-color marking)
Frequency: Ít빈 hơn (~50-200ms intervals)
Pause time: ~10-100ms (longer, but incremental)
Objects: Sống lâu, survived ≥ 2 GC cycles
Trigger: Khi Old Space đạt ~80-90% capacity
Throughput: ~2-5 MB/ms (slower than Scavenge)
```

**Mark-Sweep-Compact Phases**:
```
1. MARK phase: 
   - Start from GC roots (stack, global, handles)
   - Traverse object graph, mark reachable objects
   - Tri-color marking: White (unmarked) → Gray (marked, unscanned) → Black (marked, scanned)
   
2. SWEEP phase:
   - Scan heap, identify unmarked (white) objects
   - Add to free list
   
3. COMPACT phase (optional, khi fragmentation cao):
   - Move objects để eliminate fragmentation
   - Update all references
   - Creates contiguous free space
```

**Promotion Criteria**:
- Age threshold: Survived 2 Scavenge cycles
- Size threshold: Objects > 16KB promoted immediately (skip young gen)
- Space pressure: Young gen full → promote aggressively

### 2.3. Incremental & Concurrent GC (Orinoco Project)

**Vấn đề**: Stop-the-world (STW) pauses block application, gây dropped frames và janky UI

**Lịch sử Evolution**:
```
Old (2010s): Full STW GC, 100-500ms pauses → Unusable cho real-time apps
V8 v5.0 (2016): Incremental marking
V8 v6.0 (2017): Concurrent marking (Orinoco)
V8 v7.0+ (2018-present): Concurrent sweeping + compaction
```

**Solution 1: Incremental GC** (Break work into chunks)
```javascript
// Concept: Break GC into small steps
while (has_more_marking_work()) {
  mark_some_objects();  // Mark ~100-500 objects
  yield_to_application();  // Let app run for ~5-10ms
}

// Result:
// - Instead of 1 × 100ms pause
// - NOW: 20 × 5ms pauses (interleaved với app code)
// - Tổng time: ~120ms (20% overhead)
// - Nhưng perceived latency: 5ms instead of 100ms!
```

**Benefits**:
- Pause time giảm từ 100ms → 5-10ms
- No dropped frames (giữ được 60 FPS)
- Better user experience

**Trade-offs**:
- Tổng GC time cao hơn ~20-30% (due to write barriers)
- Complexity: Phải handle concurrent modifications

**Solution 2: Concurrent GC** (GC on separate threads)
```javascript
// Concept: GC runs parallel với application

// Main thread:           Helper threads:
// Run JavaScript         Mark objects
// No pause needed  ←     Sweep memory
//                        Compact heap

// Near-zero pause time (chỉ pause for final sync ~1-2ms)
```

**Implementation Details**:
- **Concurrent Marking**: Helper threads traverse object graph while app runs
- **Write Barriers**: Track modifications during concurrent marking
- **Final Pause**: Short STW pause (~1-2ms) để finish & sync
- **Thread Count**: Typically 2-4 helper threads (CPU cores - 1)

**Benefits**:
- Near-zero perceived pause (~1-2ms)
- Scales với CPU cores
- Smooth 60 FPS gameplay/animations possible

**Trade-offs**:
- CPU overhead: ~10-20% more total CPU time
- Memory overhead: ~5-10MB for GC state
- Complexity: Write barriers, memory barriers, synchronization

### 2.4. Write Barriers - Tracking Cross-Generation References

**Vấn đề**: Old Gen objects có thể reference Young Gen objects
```javascript
// Scenario:
const oldObject = {};  // In Old Generation (survived many GCs)
const youngObject = {}; // In Young Generation (newly created)

oldObject.child = youngObject;  // Cross-generation reference!

// Problem: Khi Scavenge Young Gen:
// - Normally chỉ scan Young Gen roots
// - Nhưng youngObject được referenced từ oldObject
// - Nếu không track → youngObject bị GC nhầm! → Crash!
```

**Write Barrier Mechanism**:
```javascript
// Pseudo-code for write barrier
function writeField(object, field, value) {
  object[field] = value;  // Actual write
  
  // WRITE BARRIER: Track cross-gen references
  if (isInOldGen(object) && isInYoungGen(value)) {
    markObjectAsDirty(object);  // Add to "remembered set"
  }
}
```

**Remembered Set** (Card Marking):
```
Old Gen chia thành "cards" (~512 bytes mỗi card)
Khi old → young reference:
- Mark card as "dirty"
- Young Gen GC scans dirty cards as additional roots
- Ngăn chặn premature collection

Memory overhead: 1 byte per 512 bytes = 0.2% memory overhead
```

**Performance Impact**:
- Write barrier cost: ~2-5 CPU cycles per write (negligible)
- Trade-off: Small write overhead for correct cross-gen GC
- Critical for generational hypothesis to work

### 2.5. GC Triggers

**Automatic triggers**:
- Young Gen full (allocation failure)
- Old Gen reaches threshold (~80%)
- Idle time (idle-time GC)

**Manual triggers** (Node.js):
```
global.gc() - Full GC
--expose-gc flag required
```

**Best practice**: Không force GC, trust the engine!

---

## 3. Memory Leaks - Phân Tích & Phòng Tránh

### 3.1. Anatomy of a Memory Leak

**Definition**: Objects không còn cần thiết nhưng vẫn **reachable** → không được GC

**Impact**:
- Memory usage tăng dần
- Performance degradation
- Out of Memory crash
- Restart cần thiết

### 3.2. Common Leak Patterns

#### Pattern 1: Accidental Globals - Unintended Global Scope Pollution

**Root Cause**: Biến không declare trong sloppy mode tạo global property

**Code Example**:
```javascript
// ❌ BAD: Creates global variable
function createLeak() {
  leakedVar = 'I am global!';  // Missing 'var/let/const'
  // Equivalent to: window.leakedVar = 'I am global!'
}

createLeak();
console.log(window.leakedVar);  // 'I am global!' - LEAKED!
```

**Impact**: **High** - Never collected, accumulates indefinitely

**Detection**:
- ESLint rule: `no-undef`, `no-implicit-globals`
- Check `Object.keys(window)` length over time

**Prevention**:
```javascript
// ✅ GOOD: Use strict mode + proper declarations
'use strict';

function noLeak() {
  const localVar = 'I am local';  // Properly scoped
  leakedVar = 'Error!';  // ReferenceError in strict mode
}

// Or use ESLint + TypeScript for compile-time checks
```

#### Pattern 2: Forgotten Timers - Zombie Callbacks

**Root Cause**: `setInterval`/`setTimeout` không clear, callbacks chạy mãi mãi

**Code Example**:
```javascript
// ❌ BAD: Timer leak in React component
class Dashboard extends React.Component {
  componentDidMount() {
    // Polling every 5s
    this.intervalId = setInterval(() => {
      this.fetchData();  // 'this' giữ reference đến component
    }, 5000);
  }
  
  // Missing cleanup! Component unmounts nhưng timer vẫn chạy
}
// Result: fetchData() calls mãi, component never GC'd
```

**Symptoms**:
- Memory tăng theo thời gian (linear growth)
- API calls tiếp tục sau component unmount
- "Cannot perform state update on unmounted component" warnings

**Proper Implementation**:
```javascript
// ✅ GOOD: Cleanup trong lifecycle
class Dashboard extends React.Component {
  componentDidMount() {
    this.intervalId = setInterval(() => {
      this.fetchData();
    }, 5000);
  }
  
  componentWillUnmount() {
    clearInterval(this.intervalId);  // CRITICAL: Cleanup!
    this.intervalId = null;
  }
}

// ✅ BETTER: React Hooks với auto-cleanup
function Dashboard() {
  useEffect(() => {
    const intervalId = setInterval(() => {
      fetchData();
    }, 5000);
    
    return () => clearInterval(intervalId);  // Cleanup function
  }, []);
}
```

**WeakMap Pattern** (for complex scenarios):
```javascript
const timers = new WeakMap();

function startTimer(component) {
  const id = setInterval(() => component.update(), 1000);
  timers.set(component, id);
}

function stopTimer(component) {
  clearInterval(timers.get(component));
  timers.delete(component);
}
// Auto-cleanup when component GC'd
```

#### Pattern 3: Event Listener Leaks - DOM Reference Retention

**Root Cause**: `addEventListener` không `removeEventListener`, giữ DOM + closure

**Leak Scenario**:
```javascript
// ❌ BAD: SPA navigation leak
class ImageGallery {
  constructor() {
    this.images = loadImages();  // Large array
    
    // Event listener giữ reference đến 'this'
    document.addEventListener('scroll', () => {
      this.handleScroll();  // Closure captures 'this'
    });
  }
  
  destroy() {
    // Missing removeEventListener!
    // Listener still active, 'this' cannot be GC'd
  }
}

// User navigates: gallery1 -> gallery2 -> gallery3
// Result: All 3 galleries + 3 scroll listeners in memory!
```

**Proper Cleanup**:
```javascript
// ✅ GOOD: Store reference and cleanup
class ImageGallery {
  constructor() {
    this.images = loadImages();
    this.handleScroll = this.handleScroll.bind(this);
    document.addEventListener('scroll', this.handleScroll);
  }
  
  destroy() {
    document.removeEventListener('scroll', this.handleScroll);
    this.images = null;  // Help GC
  }
}
```

**Modern Solution: AbortController**
```javascript
// ✅ BEST: Auto-cleanup với AbortController
class ImageGallery {
  constructor() {
    this.controller = new AbortController();
    
    document.addEventListener('scroll', () => {
      this.handleScroll();
    }, { signal: this.controller.signal });
    
    window.addEventListener('resize', () => {
      this.handleResize();
    }, { signal: this.controller.signal });
  }
  
  destroy() {
    this.controller.abort();  // Removes ALL listeners at once!
  }
}
```

**React Pattern**:
```javascript
function ImageGallery() {
  useEffect(() => {
    const handleScroll = () => { /* ... */ };
    
    document.addEventListener('scroll', handleScroll);
    return () => document.removeEventListener('scroll', handleScroll);
  }, []);
}
```

#### Pattern 4: Closure Traps - Heavy Context Retention

**Root Cause**: Closure captures toàn bộ outer scope, k cả unused large variables

**Leak Example**:
```javascript
// ❌ BAD: Closure giữ entire large scope
function processData() {
  const hugeArray = new Array(1000000).fill({data: 'value'});
  const largeObject = { /* 10MB data */ };
  
  return function getFirstItem() {
    return hugeArray[0];  // Only uses first item
    // BUT: Entire hugeArray + largeObject retained in closure!
  };
}

const getter = processData();
// hugeArray (1M items) và largeObject vẫn trong memory
// dù chỉ cần 1 item!
```

**Memory Impact**: 1 closure = 10MB+ retained unnecessarily

**Solution 1: Extract needed data**
```javascript
// ✅ GOOD: Only capture what's needed
function processData() {
  const hugeArray = new Array(1000000).fill({data: 'value'});
  const largeObject = { /* 10MB data */ };
  
  const firstItem = hugeArray[0];  // Extract before closure
  
  return function getFirstItem() {
    return firstItem;  // Only captures firstItem (tiny!)
  };
  // hugeArray & largeObject can be GC'd after function returns
}
```

**Solution 2: Nullify large variables**
```javascript
function processData() {
  let hugeArray = new Array(1000000).fill({data: 'value'});
  
  const result = hugeArray[0];
  hugeArray = null;  // Help GC immediately
  
  return () => result;
}
```

**Production Tip**: Monitor closure sizes với Chrome DevTools Heap Snapshot

#### Pattern 5: Detached DOM Nodes - Zombie Elements

**Root Cause**: JavaScript giữ reference đến removed DOM nodes

**Leak Scenario**:
```javascript
// ❌ BAD: Cache DOM references
const cache = [];

function addItem(text) {
  const item = document.createElement('div');
  item.textContent = text;
  document.body.appendChild(item);
  
  cache.push(item);  // Store reference
}

function removeAll() {
  document.body.innerHTML = '';  // DOM cleared
  // BUT: cache still holds references → Detached DOM leak!
}

addItem('Item 1');
addItem('Item 2');
removeAll();
// cache[0], cache[1] are detached but not GC'd
```

**Detection**: Chrome DevTools "Detached DOM tree" filter

**Impact**:
- Each detached node: ~100 bytes (node) + children + event listeners
- 1000 detached nodes = ~100KB+ leaked

**Solution 1: Clear references**
```javascript
// ✅ GOOD: Clear cache on removal
const cache = [];

function removeAll() {
  document.body.innerHTML = '';
  cache.length = 0;  // Clear array
}
```

**Solution 2: WeakMap (preferred)**
```javascript
// ✅ BEST: Auto-cleanup với WeakMap
const metadata = new WeakMap();

function addItem(text) {
  const item = document.createElement('div');
  item.textContent = text;
  document.body.appendChild(item);
  
  metadata.set(item, { created: Date.now() });
  // Khi item removed từ DOM → WeakMap entry auto-removed
}
```

**Solution 3: Event delegation**
```javascript
// ✅ BETTER: No individual listeners
document.body.addEventListener('click', (e) => {
  if (e.target.matches('.item')) {
    handleClick(e.target);
  }
});
// No per-element listeners → no detached listener leaks
```

#### Pattern 6: Unbounded Caches - Infinite Growth

**Root Cause**: Cache grows indefinitely without eviction policy

**Leak Example**:
```javascript
// ❌ BAD: Cache never clears
const imageCache = new Map();

function loadImage(url) {
  if (imageCache.has(url)) {
    return imageCache.get(url);
  }
  
  const img = new Image();
  img.src = url;
  imageCache.set(url, img);  // Grows forever!
  return img;
}

// After loading 10,000 images:
// imageCache size: 10,000 entries
// Memory: ~500MB (assuming 50KB per image)
// Problem: Rarely-used images never evicted
```

**Solution 1: LRU Cache với size limit**
```javascript
// ✅ GOOD: LRU (Least Recently Used) cache
class LRUCache {
  constructor(maxSize = 100) {
    this.cache = new Map();
    this.maxSize = maxSize;
  }
  
  get(key) {
    if (!this.cache.has(key)) return undefined;
    
    const value = this.cache.get(key);
    // Move to end (most recently used)
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }
  
  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }
    
    this.cache.set(key, value);
    
    // Evict oldest if exceeds max size
    if (this.cache.size > this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }
}

const imageCache = new LRUCache(100);  // Max 100 images
```

**Solution 2: TTL (Time-To-Live)**
```javascript
// ✅ GOOD: Auto-expire old entries
class TTLCache {
  constructor(ttl = 60000) {  // 60s default
    this.cache = new Map();
    this.ttl = ttl;
  }
  
  set(key, value) {
    const entry = {
      value,
      expiry: Date.now() + this.ttl
    };
    this.cache.set(key, entry);
  }
  
  get(key) {
    const entry = this.cache.get(key);
    if (!entry) return undefined;
    
    if (Date.now() > entry.expiry) {
      this.cache.delete(key);  // Expired
      return undefined;
    }
    
    return entry.value;
  }
  
  cleanup() {
    const now = Date.now();
    for (const [key, entry] of this.cache) {
      if (now > entry.expiry) {
        this.cache.delete(key);
      }
    }
  }
}

const cache = new TTLCache(60000);
setInterval(() => cache.cleanup(), 10000);  // Cleanup every 10s
```

**Solution 3: WeakMap (for object keys)**
```javascript
// ✅ BEST: Auto-cleanup khi object GC'd
const cache = new WeakMap();

function process(obj) {
  if (cache.has(obj)) {
    return cache.get(obj);
  }
  
  const result = expensiveComputation(obj);
  cache.set(obj, result);
  return result;
}
// Khi obj không còn referenced → cache entry auto-removed
```

### 3.3. Real-World Case Studies

#### Case Study 1: Gmail Memory Leak

**Scenario**: Long-running email client session

**Leak sources**:
- Cached emails (100s of MB)
- Event listeners on email items
- Undo/redo history

**Solution**:
- Virtual scrolling (only render visible)
- Periodic cache cleanup
- Limit undo history (50 actions)

#### Case Study 2: VS Code Extension Leak

**Scenario**: Language server process

**Leak source**:
- Symbol/AST cache for files
- Watchers không dispose

**Solution**:
- Per-file cache limit
- Dispose watchers on file close
- Background cache cleanup

#### Case Study 3: React SPA Navigation

**Scenario**: User navigates giữa nhiều routes

**Leak sources**:
- Global event listeners
- Subscriptions không unsubscribe
- Timers in effects

**Solution**:
- useEffect cleanup function
- Abort controllers
- Custom hooks cho cleanup

---

## 4. WeakMap & WeakSet - Advanced Use Cases

### 4.1. Weak References Explained

**Strong reference**: Prevents GC
```
const obj = { data: 'value' };
const map = new Map();
map.set(obj, 'metadata');
// obj referenced → never collected while map exists
```

**Weak reference**: Allows GC
```
const obj = { data: 'value' };
const weakMap = new WeakMap();
weakMap.set(obj, 'metadata');
// obj not referenced elsewhere → can be collected
// → weakMap entry automatically removed
```

### 4.2. WeakMap Use Cases

#### Use Case 1: Private Data Storage

**Vấn đề**: JavaScript trước ES2022 không có native private fields

**Solution**: WeakMap as private storage
```javascript
// ✅ WeakMap-based private fields
const privateData = new WeakMap();

class BankAccount {
  constructor(balance) {
    privateData.set(this, { balance });  // Truly private!
  }
  
  getBalance() {
    return privateData.get(this).balance;
  }
  
  deposit(amount) {
    const data = privateData.get(this);
    data.balance += amount;
  }
}

const account = new BankAccount(1000);
console.log(account.balance);  // undefined - cannot access!
console.log(account.getBalance());  // 1000 - only via methods

// Auto-cleanup:
account = null;  // privateData entry auto-removed!
```

**Benefits**:
- Truly private (không access được từ outside)
- Auto-cleanup khi instance GC'd
- No manual cleanup needed

**Limitations**: Cannot iterate, no `.size`

#### Use Case 2: DOM Metadata Storage

**Vấn đề**: Attach metadata to DOM nodes, auto-cleanup khi removed

**Solution**: WeakMap với DOM keys
```javascript
// ✅ Analytics tracking với auto-cleanup
const elementMetadata = new WeakMap();

function trackElement(element, eventName) {
  if (!elementMetadata.has(element)) {
    elementMetadata.set(element, {
      clicks: 0,
      firstSeen: Date.now(),
      events: []
    });
  }
  
  const metadata = elementMetadata.get(element);
  metadata.clicks++;
  metadata.events.push({ event: eventName, time: Date.now() });
}

// Usage:
const button = document.querySelector('#submit');
button.addEventListener('click', () => {
  trackElement(button, 'click');
  console.log(elementMetadata.get(button));  // { clicks: 1, ... }
});

// Khi button removed từ DOM:
button.remove();
// → WeakMap entry auto-removed, no manual cleanup!
```

**Use Cases**:
- Component instances tracking
- Event delegation state
- Analytics click/view data
- UI state management

**Benefits**:
- Zero memory leaks (auto-cleanup)
- No manual cleanup code needed
- Perfect cho SPA navigation

#### Use Case 3: Memoization Cache

**Vấn đề**: Cache expensive function results, auto-cleanup unused

**Implementation**:
```javascript
// ✅ WeakMap memoization
const memoCache = new WeakMap();

function expensiveComputation(obj) {
  if (memoCache.has(obj)) {
    console.log('Cache hit!');
    return memoCache.get(obj);
  }
  
  console.log('Computing...');
  // Expensive computation
  const result = Object.keys(obj).reduce((sum, key) => {
    return sum + (typeof obj[key] === 'number' ? obj[key] : 0);
  }, 0);
  
  memoCache.set(obj, result);
  return result;
}

// Usage:
const data1 = { a: 1, b: 2, c: 3 };
expensiveComputation(data1);  // "Computing..." -> 6
expensiveComputation(data1);  // "Cache hit!" -> 6

data1 = null;  // Cache entry auto-removed!
```

**Benefits**:
- Auto-cleanup cho unused objects
- No memory leak risk
- Zero manual cache management

**Limitations**:
- Chỉ works với object arguments (not primitives)
- Cannot clear entire cache manually
- Cannot inspect cache size

#### Use Case 4: Object Capability Pattern

**Concept**: WeakMap as capability token

**Pattern**:
```
WeakMap stores permissions/capabilities
Object as key = capability holder
No key = no access
Auto-revokes when object GC'd
```

### 4.3. WeakSet Use Cases

#### Use Case 1: Visited Tracking

**Scenario**: Track processed/visited objects

**Benefits**:
- Auto-cleanup
- No repeated processing
- Cycle detection

#### Use Case 2: Object Marking

**Problem**: Mark objects without modifying them

**Solution**: WeakSet membership
```
Use cases:
- Freeze/seal tracking
- Validation state
- Feature flags
```

### 4.4. Limitations & Pitfalls

**WeakMap/WeakSet cannot**:
- Iterate (no .keys(), .values(), .entries())
- Get size (no .size property)
- Clear all (no .clear() method)
- Use primitive keys (objects only)

**Why?**: Weak references are non-deterministic

**When NOT to use**:
- Need iteration
- Need enumeration
- Cannot use object keys
- Need guaranteed lifetime

---

## 5. Chrome DevTools - Profiling Techniques

### 5.1. Memory Profiling Tools

#### Tool 1: Heap Snapshot
```
What: Point-in-time memory state
When: Compare states, find leaks
How: Take multiple snapshots, compare
```

#### Tool 2: Allocation Timeline
```
What: Record allocations over time
When: Find memory growth patterns
How: Analyze blue bars (allocations)
```

#### Tool 3: Allocation Sampling
```
What: Statistical sampling
When: Low overhead profiling
How: Sample-based, less detailed
```

### 5.2. Heap Snapshot Analysis

#### View 1: Summary
```
Purpose: See objects grouped by constructor
Use: Find unexpected object counts
Look for: Large retained size
```

#### View 2: Comparison
```
Purpose: Compare two snapshots
Use: Find memory leaks
Look for: Objects added but not removed
```

#### View 3: Containment
```
Purpose: Object reference tree
Use: Understand retention paths
Look for: Why object not GC'd
```

#### View 4: Statistics
```
Purpose: Memory distribution
Use: Pie chart của types
Look for: Unexpected large categories
```

### 5.3. Finding Memory Leaks - 3-Snapshot Technique

**Mục tiêu**: Identify objects không được GC sau action

**Steps chi tiết**:
```
1. Snapshot 1 (Baseline): 
   - Load page
   - Take heap snapshot
   - Note memory: ~50MB

2. Action (Suspected leak):
   - Open modal dialog
   - Interact with UI
   - Memory now: ~60MB (+10MB)

3. Snapshot 2 (After action):
   - Take heap snapshot
   - Memory: ~60MB

4. Reverse Action (Cleanup):
   - Close modal dialog  
   - Force GC (Chrome: DevTools -> Performance Monitor -> Collect Garbage)
   - Wait 10s

5. Snapshot 3 (After cleanup):
   - Take heap snapshot
   - Memory: ~58MB (không về 50MB!)

6. Compare Snapshots:
   - Snapshot 3 vs Snapshot 1
   - Delta: +8MB leaked!
```

**Expected Result**: Snapshot 3 ≈ Snapshot 1 (tolerance: ±2MB)

**Leak Indicator**: Snapshot 3 >> Snapshot 1

**Analysis**:
```
Comparison view shows:
- HTMLDivElement: +500 instances (modal elements not cleaned)
- EventListener: +50 instances (listeners not removed)
- Closure: +200 instances (callbacks not cleared)

Retainer path:
  Window -> eventListeners -> onClick -> <closure> -> modalComponent
  
Fix: Remove event listeners trong modal.destroy()
```

#### Detached DOM Analysis

**Filter**: "Detached" in Summary view

**Look for**:
- HTMLDivElement (detached)
- Event listeners on detached nodes
- Retaining paths từ JavaScript

**Fix**: Remove references hoặc listeners

### 5.4. Performance Monitor - Real-time Metrics

**Critical Metrics to Track**:

#### 1. JS Heap Size (Live Objects)
```
Normal: Sawtooth pattern (allocate -> GC -> allocate)
  /\      /\      /\
 /  \    /  \    /  \
/    \  /    \  /    \
      \/      \/

Leak: Ascending baseline
    /\      /\      /\
   /  \    /  \    /  \
  /    \  /    \  /    \
 /      \/      \/
```

**Red Flag**: Baseline tăng dần sau mỗi GC cycle

#### 2. DOM Nodes Count
```
Normal: Stable (~500-2000 nodes for typical page)
Leak: Increasing indefinitely (grows by 100s per action)
```

**Investigation**:
- Filter "Detached" trong Heap Snapshot
- Look for large counts: `HTMLDivElement (detached)`, `HTMLSpanElement (detached)`

#### 3. Event Listeners Count
```
Normal: Proportional to DOM nodes (~1-2 per interactive element)
Leak: Grows faster than DOM nodes

Example:
- Page load: 50 listeners
- After 10 navigations: 500 listeners (!)
- Expected: ~50 listeners (stable)
```

**Fix**: Implement AbortController pattern

#### 4. Documents & Frames Count
```
Normal: 1 document (single-page app)
Leak: Multiple documents retained (iframe leaks)
```

**Detection Tool**: Performance Monitor tab trong Chrome DevTools
- Open: Cmd+Shift+P (Mac) / Ctrl+Shift+P (Windows) -> "Show Performance Monitor"
- Monitor real-time while using app
- Look for trends over 30-60s

---

## 6. Production Optimization Strategies

### 6.1. Object Pooling

**Concept**: Reuse objects instead of create/destroy

**When to use**:
- High-frequency object creation
- Uniform objects (particles, bullets)
- GC pressure observable

**Benefits**:
- Reduce GC frequency
- Predictable performance
- Lower memory churn

**Trade-offs**:
- Memory overhead (pool size)
- Complexity
- Cache pollution

**Use cases**:
- Game engines (particles, entities)
- Canvas/WebGL rendering
- Event objects

### 6.2. Lazy Initialization

**Concept**: Defer object creation until needed

**Patterns**:

#### Pattern 1: Lazy Properties
```
Getter creates on first access
Subsequent accesses use cached
```

#### Pattern 2: Lazy Loading
```
Load modules/data on demand
Reduce initial bundle size
```

**Benefits**:
- Faster startup
- Lower initial memory
- Better user experience

### 6.3. Memory Pooling Strategies

#### Strategy 1: Pre-allocation
```
Allocate pool upfront
Trade: Memory for performance
Use: Predictable load
```

#### Strategy 2: Dynamic Growth
```
Start small, grow as needed
Trade: Performance for memory
Use: Variable load
```

#### Strategy 3: Hybrid
```
Initial pool + dynamic growth + max cap
Balance memory and performance
```

### 6.4. Data Structure Selection

**Performance vs Memory trade-offs**:

#### Array vs Object
```
Array: 
- Faster iteration
- Lower memory (packed elements)
- Use: Homogeneous data, iteration

Object:
- Faster lookup
- Higher memory (hash table)
- Use: Key-value, random access
```

#### Map vs Object
```
Map:
- Better for frequent add/remove
- Any type as key
- Maintains insertion order

Object:
- Better for static keys
- Prototype chain lookup
- JSON serializable
```

#### Set vs Array (uniqueness)
```
Set:
- O(1) lookup/add/delete
- Automatic uniqueness

Array + includes():
- O(n) lookup
- Manual dedup needed
```

### 6.5. Production Monitoring - Metrics & Alerts

**Mục tiêu**: Detect memory issues trước khi crash

#### 1. Heap Size Trends
```javascript
// Node.js monitoring
const v8 = require('v8');

setInterval(() => {
  const heapStats = v8.getHeapStatistics();
  const usedHeap = heapStats.used_heap_size / heapStats.heap_size_limit;
  
  console.log(`Heap usage: ${(usedHeap * 100).toFixed(2)}%`);
  
  // Alert conditions:
  if (usedHeap > 0.9) {
    logger.critical('Heap near limit!', heapStats);
  } else if (usedHeap > 0.8) {
    logger.warn('Heap usage high', heapStats);
  }
}, 60000);  // Check every minute
```

**Alert Thresholds**:
- **Warning**: >80% heap usage
- **Critical**: >90% heap usage
- **Action**: Investigate leaks, increase limits, or restart

#### 2. GC Frequency Monitoring
```javascript
// Track GC events
const { PerformanceObserver } = require('perf_hooks');

const gcObserver = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`GC ${entry.kind}: ${entry.duration}ms`);
    
    if (entry.duration > 100) {
      logger.warn('Long GC pause', { duration: entry.duration });
    }
  });
});

gcObserver.observe({ entryTypes: ['gc'] });
```

**Healthy GC Pattern**:
- Young Gen: 5-20 times/minute, <5ms each
- Old Gen: 1-5 times/hour, <50ms each

**Unhealthy Patterns**:
- Too frequent: >50 Young Gen GCs/minute -> High allocation rate
- Too rare: <1 GC/hour -> Possible memory bloat
- Long pauses: >100ms -> Too many live objects

#### 3. Memory Limit Proximity
```javascript
function checkMemoryHealth() {
  const usage = process.memoryUsage();
  const heapUsedMB = usage.heapUsed / 1024 / 1024;
  const heapTotalMB = usage.heapTotal / 1024 / 1024;
  const externalMB = usage.external / 1024 / 1024;
  
  return {
    heapUsedMB: heapUsedMB.toFixed(2),
    heapTotalMB: heapTotalMB.toFixed(2),
    externalMB: externalMB.toFixed(2),
    heapUsagePercent: ((heapUsedMB / heapTotalMB) * 100).toFixed(2)
  };
}

setInterval(() => {
  const health = checkMemoryHealth();
  metrics.gauge('memory.heap.used', health.heapUsedMB);
  metrics.gauge('memory.heap.total', health.heapTotalMB);
  
  if (health.heapUsagePercent > 90) {
    alerting.trigger('memory-critical', health);
  }
}, 30000);
```

#### 4. APM Integration
```javascript
// Example: Datadog APM
const tracer = require('dd-trace').init();

// Auto-tracks:
// - Heap size over time
// - GC pause times
// - Memory allocation rates
// - Slow memory warnings

// Custom metrics:
const { dogstatsd } = tracer;
setInterval(() => {
  const stats = v8.getHeapStatistics();
  dogstatsd.gauge('app.heap.size', stats.used_heap_size);
  dogstatsd.gauge('app.heap.limit', stats.heap_size_limit);
}, 10000);
```

**Recommended APM Tools**:
- **Datadog**: Excellent V8 heap tracking
- **New Relic**: Good memory profiling
- **Dynatrace**: AI-powered leak detection
- **Elastic APM**: Open-source alternative

### 6.6. Memory Budgets

**Concept**: Set memory limits per feature/route

**Example budgets**:
```
Initial load: <5 MB JS heap
Per route: <2 MB additional
Cache: <10 MB total
Media: <50 MB total
```

**Enforcement**:
- Webpack bundle analyzer
- Performance budgets in CI
- Runtime monitoring

---

## 📝 Key Takeaways

### Memory Model
- Stack: Fast, automatic, limited
- Heap: Large, GC-managed, slower
- Hidden classes: Critical for performance
- Inline caching: V8 optimization

### Garbage Collection
- Generational: Young + Old generations
- Incremental & Concurrent: Reduce pauses
- Mark-Sweep-Compact: Foundation
- Write barriers: Track cross-gen refs

### Memory Leaks
- 6 common patterns: Globals, timers, listeners, closures, DOM, caches
- Real-world examples: Gmail, VS Code, React SPAs
- Detection: 3-snapshot technique
- Prevention: Cleanup, WeakMap, limits

### WeakMap/WeakSet
- Weak references allow GC
- Use cases: Private data, DOM metadata, memoization, tracking
- Limitations: No iteration, objects only
- Auto-cleanup benefit

### Profiling
- Heap snapshots: Find leaks
- Allocation timeline: Growth patterns
- 4 views: Summary, Comparison, Containment, Statistics
- Detached DOM: Common leak source

### Optimization
- Object pooling: Reduce GC
- Lazy initialization: Defer allocation
- Data structure selection: Performance vs memory
- Monitoring: Track trends, set budgets
- Production metrics: Heap, GC, limits

---

## 🎯 Advanced Topics

1. **Off-heap memory** (ArrayBuffer, TypedArrays)
2. **Shared memory** (SharedArrayBuffer)
3. **Memory compression** (V8 pointer compression)
4. **Custom allocators** (WebAssembly linear memory)
5. **Memory snapshots** (Heap dumps analysis)
6. **GC tuning** (Node.js flags: --max-old-space-size)

---

**Chúc bạn học tốt! 🚀**
