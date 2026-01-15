# JavaScript Asynchronous Programming - Từ Lý Thuyết Đến Thực Chiến

> **Mục tiêu**: Hiểu sâu về cơ chế bất đồng bộ trong JavaScript từ internals đến production patterns, với focus vào practical applications và performance.

---

## 📚 Table of Contents

1. [Event Loop Architecture](#1-event-loop-architecture)
2. [Microtask vs Macrotask - Phân Biệt & Ứng Dụng](#2-microtask-vs-macrotask---phân-biệt--ứng-dụng)
3. [Promises - Internals & Production Patterns](#3-promises---internals--production-patterns)
4. [Async/Await - Advanced Techniques](#4-asyncawait---advanced-techniques)
5. [Advanced Async Patterns](#5-advanced-async-patterns)
6. [Async Iteration & Generators](#6-async-iteration--generators)
7. [Error Handling in Production](#7-error-handling-in-production)
8. [Performance Optimization](#8-performance-optimization)
9. [Testing Async Code](#9-testing-async-code)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. Event Loop Architecture

### 1.1. Tại Sao JavaScript Cần Event Loop?

**Vấn đề cốt lõi**: JavaScript chạy trên mô hình **single-threaded** (đơn luồng), nhưng phải xử lý đồng thời nhiều tác vụ:
- Tương tác người dùng (clicks, scrolls, keyboard input)
- Network I/O (fetch API, XMLHttpRequest)
- Timers (setTimeout, setInterval, requestAnimationFrame)
- File system operations (Node.js fs module, File API)
- DOM rendering và layout calculations

**Vấn đề nếu chạy tuần tự**: Bất kỳ **long-running synchronous operations** (CPU-intensive tasks, synchronous I/O, blocking loops) đều block main thread, dẫn đến UI lag nghiêm trọng với frame drops dưới 60 FPS.

**Giải pháp kiến trúc**: Event Loop điều phối execution model bất đồng bộ, offload I/O operations sang OS-level async APIs và CPU-intensive tasks sang Web Workers.

### 1.2. Kiến Trúc Runtime của JavaScript

```
┌───────────────────────────┐
│   JavaScript Engine       │
│  ┌─────────────────────┐  │
│  │    Call Stack       │  │  (LIFO - thực thi synchronous code)
│  └─────────────────────┘  │
│  ┌─────────────────────┐  │
│  │    Memory Heap      │  │  (Dynamic allocation)
│  └─────────────────────┘  │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│   Browser/Node APIs       │
│  - setTimeout             │
│  - fetch                  │
│  - DOM Events             │
│  - File I/O (Node)        │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  Microtask Queue          │ ← Độ ưu tiên CAO
│  - Promise.then/catch     │
│  - queueMicrotask()       │
│  - MutationObserver       │
│  - process.nextTick()     │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  Macrotask Queue          │ ← Độ ưu tiên THẤP
│  (Task Queue)             │
│  - setTimeout             │
│  - setInterval            │
│  - setImmediate           │
│  - I/O callbacks          │
│  - UI rendering           │
└─────────────┬─────────────┘
              │
              ▼
       ┌──────────────┐
       │  Event Loop  │
       └──────────────┘
```

**3 thành phần chính**:

1. **JavaScript Engine** (V8, SpiderMonkey, JavaScriptCore):
   - **Call Stack**: Cấu trúc LIFO quản lý execution contexts, xử lý các operations đồng bộ tuần tự
   - **Memory Heap**: Vùng nhớ non-contiguous cho dynamic object allocation, được quản lý bởi garbage collector

2. **Web APIs / Node.js C++ Bindings**:
   - **Browser Runtime**: DOM APIs, Fetch API, Timer APIs, IndexedDB, WebCrypto
   - **Node.js Runtime**: libuv-backed modules (fs, http, crypto), native timers
   - **Nguyên lý quan trọng**: Offload I/O operations sang OS-level async APIs, giải phóng main thread

3. **Event Loop**:
   - **Cơ chế điều phối**: Liên tục monitor call stack và task queues, orchestrate task execution
   - **Thuật toán scheduling**: Ưu tiên microtasks trước macrotasks, đảm bảo execution order xác định
   - **Performance characteristic**: ~1ms mỗi iteration trong điều kiện bình thường

### 1.3. Tại Sao Có 2 Loại Queues?

#### 1️⃣ Microtask Queue → High-Priority Atomic Operations

**Đặc điểm thực thi**:
- **Đảm bảo hoàn thành nguyên tử**: Tất cả microtasks trong queue được thực thi tuần tự trước khi yield control cho browser rendering hoặc macrotask processing
- **Use cases chính**:
  - Promise resolution/rejection callbacks (.then, .catch, .finally)
  - Async function continuation sau await points
  - Framework internal state reconciliation (Virtual DOM diffing)
  - MutationObserver callbacks
  - queueMicrotask() API

**Lý do thực thi theo batch**:
- **Tính nhất quán state**: Ngăn intermediate UI states bị render trong quá trình multi-step state transitions
- **Tối ưu rendering**: Giảm layout thrashing không cần thiết bằng cách batch DOM reads/writes
- **Performance impact**: Chỉ invoke render pipeline một lần thay vì nhiều lần partial renders
- **Ví dụ**: React batch state updates trong event handlers để trigger single re-render

**Ý nghĩa quan trọng**: Microtask queue phải complete atomically để duy trì synchronous appearance của JavaScript trong async boundaries.

#### 2️⃣ Macrotask Queue → Deferrable Scheduled Work

**Đặc điểm thực thi**:
- **Single-Task-Per-Tick**: Event loop xử lý đúng một macrotask mỗi iteration, sau đó yield control
- **Interleaving Points**: Browser có thể execute rendering pipeline giữa các macrotasks:
  - Style calculation và layout (Recalc Style/Layout trong DevTools)
  - Paint và composite operations
  - User input event processing
  - requestAnimationFrame callbacks

**Lý do thiết kế**:
- **Ngăn main thread starvation**: Đảm bảo browser responsiveness bằng cách giới hạn continuous JavaScript execution
- **Bảo toàn rendering budget**: Duy trì 60 FPS target bằng cách giữ frame budget dưới ~16.67ms
- **Input responsiveness**: Cho phép xử lý user interactions trong 100ms RAIL threshold
- **Ví dụ**: setTimeout callbacks execute từng cái một, với render opportunities giữa mỗi callback

**Nguồn macrotask phổ biến**: setTimeout, setInterval, setImmediate (Node.js), I/O callbacks, UI events

#### 🎯 Tóm tắt Ưu tiên Queue:
```
Microtasks: Atomic batch execution, high priority, state-critical operations
Macrotasks: Singular execution với yield points, cho phép rendering interleaving
```

> **⚠️ Lưu ý quan trọng**: Microtask Queue và Macrotask Queue là **hai hàng đợi hoàn toàn tách biệt**, không phải cùng nằm trong một "Task Queues". Event Loop sẽ ưu tiên xử lý **TẤT CẢ** microtasks trước khi chuyển sang bất kỳ macrotask nào.

### 1.4. Event Loop Hoạt Động Như Thế Nào?

Event Loop là một **vòng lặp vô hạn** liên tục kiểm tra và thực thi các task theo thứ tự ưu tiên:

```javascript
// Pseudocode của Event Loop
while (true) {
  // 1. Kiểm tra Call Stack có rỗng không?
  if (callStack.isEmpty()) {
    
    // 2. Ưu tiên xử lý TẤT CẢ Microtasks trước
    while (microtaskQueue.hasTask()) {
      const microtask = microtaskQueue.dequeue();
      callStack.push(microtask);
      execute(microtask);
    }
    
    // 3. Sau đó mới xử lý MỘT Macrotask
    if (macrotaskQueue.hasTask()) {
      const macrotask = macrotaskQueue.dequeue();
      callStack.push(macrotask);
      execute(macrotask);
    }
    
    // 4. Render UI (nếu cần - trong browser)
    if (needsRendering) {
      renderUI();
    }
  }
}
```

### 1.5. Event Loop Phases (Node.js)

Node.js Event Loop có **6 phases**:

```
   ┌───────────────────────┐
┌─>│        timers         │  setTimeout, setInterval
│  └─────────┬─────────────┘
│  ┌─────────┴─────────────┐
│  │  pending callbacks    │  I/O callbacks từ previous iteration
│  └─────────┬─────────────┘
│  ┌─────────┴─────────────┐
│  │    idle, prepare      │  Internal use only
│  └─────────┬─────────────┘
│  ┌─────────┴─────────────┐
│  │        poll           │  Retrieve I/O events, execute callbacks
│  └─────────┬─────────────┘
│  ┌─────────┴─────────────┐
│  │        check          │  setImmediate callbacks
│  └─────────┬─────────────┘
│  ┌─────────┴─────────────┐
└──│   close callbacks     │  socket.on('close', ...)
   └───────────────────────┘
```

**Giữa mỗi phase**: Process **TẤT CẢ** Microtasks

### 1.6. Ví Dụ Minh Họa Event Loop

#### Ví dụ cơ bản:

```javascript
console.log('1: Script start');

setTimeout(() => {
  console.log('2: setTimeout');
}, 0);

Promise.resolve()
  .then(() => {
    console.log('3: Promise 1');
  })
  .then(() => {
    console.log('4: Promise 2');
  });

console.log('5: Script end');

// Output:
// 1: Script start
// 5: Script end
// 3: Promise 1
// 4: Promise 2
// 2: setTimeout
```

**Giải thích từng bước**:
1. "1: Script start" - Synchronous code, chạy ngay lập tức
2. setTimeout - Được đưa vào **Macrotask Queue**
3. Promise.resolve().then() - Được đưa vào **Microtask Queue**
4. "5: Script end" - Synchronous code, chạy ngay lập tức
5. Call Stack rỗng → Event Loop kiểm tra
6. **Microtask Queue** được xử lý trước → "3: Promise 1", "4: Promise 2"
7. **Macrotask Queue** được xử lý sau → "2: setTimeout"

#### Ví dụ phức tạp hơn:

```javascript
console.log('Start');

setTimeout(() => {
  console.log('Timeout 1');
  Promise.resolve().then(() => console.log('Promise in Timeout 1'));
}, 0);

Promise.resolve()
  .then(() => {
    console.log('Promise 1');
    setTimeout(() => console.log('Timeout in Promise 1'), 0);
  })
  .then(() => {
    console.log('Promise 2');
  });

setTimeout(() => {
  console.log('Timeout 2');
}, 0);

console.log('End');

// Output:
// Start
// End
// Promise 1
// Promise 2
// Timeout 1
// Promise in Timeout 1
// Timeout in Promise 1
// Timeout 2
```

### 1.7. Real-World Use Cases

#### **Use Case 1: Batching DOM Updates để Tối ưu Rendering**

**Vấn đề**: State updates đồng bộ naive trigger cascading re-renders
```
setState() → commit phase → layout → paint (16ms)
setState() → commit phase → layout → paint (16ms)  
setState() → commit phase → layout → paint (16ms)
Tổng: 48ms, 3 frame drops tại 60 FPS
```

**Giải pháp**: Microtask-based update batching (React 18 Automatic Batching)
```
setState() → schedule microtask (defer reconciliation)
setState() → coalesce vào pending batch
setState() → coalesce vào pending batch
→ Single commit phase sau khi microtask queue drain
Tổng: ~16ms, zero frame drops
```

**Performance Gain**: Giảm 3x render overhead thông qua batch reconciliation

#### **Use Case 2: Xử Lý Race Condition trong Search Autocomplete**

**Scenario**: Chuỗi keystroke nhanh tạo ra concurrent requests
- Keystroke "h" → Request A (network latency: 150ms)
- Keystroke "he" → Request B (network latency: 100ms)
- Keystroke "hel" → Request C (network latency: 80ms)

**Vấn đề**: Thứ tự complete non-deterministic vi phạm result freshness invariant
- Request C complete trước (kết quả đúng cho "hel")
- Request A complete cuối (stale results cho "h" override UI đúng)

**Chiến lược giải quyết**:
1. **AbortController Pattern**: Cancel in-flight requests khi query mới được issue (preferred)
2. **Request Token Validation**: Discard responses không match latest query ID
3. **Debounced Dispatch**: Defer request đến khi 300ms idle period (giảm request volume)

#### **Use Case 3: Long Task Chunking để Duy Trì Responsiveness**

**Vấn đề**: Xử lý 1M records đồng bộ blocks main thread trong 5000ms
- Frame budget tại 60 FPS: 16.67ms
- 5000ms blockage = 300 dropped frames
- Time to Interactive (TTI): Severely degraded

**Giải pháp**: Time-slicing với macrotask scheduling
```javascript
// Chunk size: 10K items × 5ms processing = 50ms per chunk
// Tổng chunks: 100
// Yield points: 99 (setTimeout giữa các chunks)

async function processLargeDataset(items) {
  const chunkSize = 10000;
  
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    processChunk(chunk);
    
    // Yield to event loop
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}

// → Browser render opportunities mỗi 50ms
// → Input responsiveness maintained dưới 100ms threshold
// → Tổng thời gian: ~5100ms (2% overhead), nhưng UI vẫn interactive
```

**Performance Trade-off**: Tăng minimal total time (scheduling overhead) đổi lại perceived performance improvement dramatic

---

## 2. Microtask vs Macrotask - Phân Biệt & Ứng Dụng

### 2.1. Sự Khác Biệt Cốt Lõi

| Aspect | Microtask | Macrotask |
|--------|-----------|-----------|
| **Timing** | Ngay sau Call Stack rỗng | Sau khi TẤT CẢ microtasks xong |
| **Quantity** | Xử lý TẤT CẢ trong queue | Xử lý MỘT task mỗi lần |
| **Priority** | Cao hơn | Thấp hơn |
| **Use Case** | State updates, Promise callbacks | User events, timers, I/O |
| **Render** | Trước khi render | Có thể render giữa các tasks |

### 2.2. Microtask Queue Sources

**Promise callbacks**:
```javascript
Promise.resolve()
  .then(() => {})    // Microtask
  .catch(() => {})   // Microtask
  .finally(() => {}); // Microtask
```

**queueMicrotask() API**:
```javascript
queueMicrotask(() => {
  console.log('Explicit microtask');
});
```

**MutationObserver** (Browser):
```javascript
const observer = new MutationObserver(() => {
  console.log('DOM mutation detected'); // Microtask
});
```

**process.nextTick()** (Node.js - highest priority):
```javascript
process.nextTick(() => {
  console.log('Next tick'); // Execute trước Promise microtasks
});
```

### 2.3. Process.nextTick - Maximum Priority Microtask (Node.js)

**Phân biệt quan trọng**: `process.nextTick()` execute trước standard microtask queue, tạo dedicated queue riêng

**Thứ tự Ưu tiên Execution** (Node.js):
```
1. process.nextTick() queue       ← Execute trước tiên, ngay sau current operation
2. Promise microtask queue        ← .then/.catch/.finally callbacks
3. queueMicrotask()               ← Explicit microtask scheduling
4. Macrotask queues               ← setTimeout/setImmediate/I/O callbacks
```

**Use Cases phù hợp**:
- **API Callback Guarantee**: Đảm bảo callbacks execute asynchronously ngay cả khi data sẵn có immediately
- **Event Emitter Pattern**: Cho phép listeners được register trước khi event emission
- **Recursive Operations**: Phải complete trước I/O events (ví dụ: recursive directory traversal)

**Cảnh báo nghiêm trọng**: Recursive `nextTick()` tạo **microtask starvation** - blocks I/O vô thời hạn

```javascript
// ❌ NGUY HIỂM: Starvation pattern
function recurse() {
  process.nextTick(recurse); // Infinite loop, I/O không bao giờ execute
}

// ✅ AN TOÀN: Cho phép I/O giữa các iterations
function recurse() {
  setImmediate(recurse); // Yields to I/O sau mỗi iteration
}
```

### 2.4. QueueMicrotask API - Standard Microtask Scheduling

**Mục đích API**: Schedule microtask execution không cần Promise wrapper overhead

**So sánh Performance**:
```javascript
// Promise-based: Tạo Promise object (allocation overhead)
Promise.resolve().then(callback); // ~10-20% chậm hơn

// queueMicrotask: Direct queue insertion (minimal overhead)
queueMicrotask(callback);          // Nhanh hơn, không object allocation
```

**Kết quả Benchmark** (V8 engine):
- `Promise.resolve().then()`: ~0.05ms per invocation
- `queueMicrotask()`: ~0.04ms per invocation  
- **Cải thiện performance 20%** trong tight loops

**Khi nào nên dùng**:
- **High-frequency scheduling**: Performance-critical paths với frequent microtask creation
- **Non-Promise workflows**: Khi resolve/reject semantics không cần thiết
- **Library internals**: Framework code cần guaranteed microtask timing

**Khi nào nên tránh**: Code hưởng lợi từ Promise composability (chaining, error handling)

### 2.5. Macrotask Queue Sources

**setTimeout / setInterval**:
```javascript
setTimeout(() => {
  console.log('Macrotask');
}, 0);
```

**setImmediate** (Node.js):
```javascript
setImmediate(() => {
  console.log('Check phase macrotask');
});
```

**I/O callbacks**:
```javascript
fs.readFile('file.txt', (err, data) => {
  console.log('I/O macrotask');
});
```

**UI Events** (Browser):
```javascript
button.addEventListener('click', () => {
  console.log('Event macrotask');
});
```

### 2.6. Pitfall Nghiêm Trọng: Microtask Starvation

**Nguyên nhân gốc**: Self-perpetuating microtask generation ngăn chặn event loop progression

**Cơ chế Starvation**:
1. **Recursive Promise Chains**: Mỗi `.then()` schedule microtask khác indefinitely
2. **MutationObserver Feedback Loop**: Observer callback triggers DOM mutation, re-queuing chính nó
3. **Framework State Cascades**: setState triggering watchers mà lại setState

**Triệu chứng quan sát được**:
- **I/O Blockage**: Network requests, file operations, timers không bao giờ execute
- **Rendering Freeze**: Browser không thể paint frames (0 FPS)
- **Unresponsive UI**: Input events được queued nhưng không bao giờ processed
- **DevTools Indication**: Long yellow scripting blocks trong Performance timeline

**Ví dụ: Promise Recursion Starvation**
```javascript
// ❌ Tạo infinite microtask loop
function starve() {
  Promise.resolve().then(() => {
    console.log('Microtask');
    starve(); // Immediately queues microtask khác
  });
}
starve();
// Kết quả: Logs flood console, UI hoàn toàn frozen

// ✅ Cho phép event loop progression
function safe() {
  setTimeout(() => {
    console.log('Macrotask');
    safe(); // Queues as macrotask, cho phép rendering giữa iterations
  }, 0);
}
```

**Chiến lược Mitigation**:

1. **Depth Limiting**: Track recursion depth, switch sang macrotask sau threshold
```javascript
let depth = 0;
const MAX_DEPTH = 100;

function process() {
  if (depth++ < MAX_DEPTH) {
    queueMicrotask(process); // Fast path
  } else {
    depth = 0;
    setTimeout(process, 0); // Yield to event loop
  }
}
```

2. **Macrotask Yielding**: Explicitly break chains với setTimeout(fn, 0)
3. **Monitoring**: Detect long microtask queues via Performance Observer

### 2.7. Ví Dụ So Sánh Chi Tiết

```javascript
console.log('1: Start');

// Macrotask
setTimeout(() => console.log('2: setTimeout'), 0);

// Microtask
Promise.resolve().then(() => console.log('3: Promise'));

// Microtask (Node.js - ưu tiên cao nhất)
process.nextTick(() => console.log('4: nextTick'));

// Microtask
queueMicrotask(() => console.log('5: queueMicrotask'));

console.log('6: End');

// Output (Node.js):
// 1: Start
// 6: End
// 4: nextTick          ← Cao nhất trong Microtask
// 3: Promise           ← Microtask
// 5: queueMicrotask    ← Microtask
// 2: setTimeout        ← Macrotask
```

**Phân tích chi tiết**:

| Bước | Call Stack | Microtask Queue | Macrotask Queue | Console Output |
|------|-----------|-----------------|-----------------|----------------|
| 1 | `console.log('1: Start')` | [] | [] | "1: Start" |
| 2 | - | [] | [`setTimeout`] | - |
| 3 | - | [`Promise handler`] | [`setTimeout`] | - |
| 4 | - | [`nextTick`, `Promise handler`] | [`setTimeout`] | - |
| 5 | - | [`nextTick`, `Promise`, `queueMicrotask`] | [`setTimeout`] | - |
| 6 | `console.log('6: End')` | [`nextTick`, `Promise`, `queueMicrotask`] | [`setTimeout`] | "6: End" |
| 7 | `nextTick handler` | [`Promise`, `queueMicrotask`] | [`setTimeout`] | "4: nextTick" |
| 8 | `Promise handler` | [`queueMicrotask`] | [`setTimeout`] | "3: Promise" |
| 9 | `queueMicrotask handler` | [] | [`setTimeout`] | "5: queueMicrotask" |
| 10 | `setTimeout handler` | [] | [] | "2: setTimeout" |

### 2.8. Real-World Use Cases

#### Use Case 1: React Concurrent Mode

React sử dụng **scheduler** để:
- High priority updates → Microtasks (user input)
- Low priority updates → Macrotasks (background updates)
- Cho phép interrupt long tasks để render urgent updates

```javascript
// React internally uses scheduler
// High priority: User input
scheduleCallback(ImmediatePriority, handleInput);

// Low priority: Background data fetch
scheduleCallback(IdlePriority, fetchData);
```

#### Use Case 2: Database Transaction Batching

**Vấn đề**: Multiple DB updates → multiple round-trips

**Giải pháp**: Batch với microtasks
```javascript
const pendingUpdates = [];

function update(data) {
  pendingUpdates.push(data);
  
  // Schedule batch processing if not already scheduled
  if (pendingUpdates.length === 1) {
    queueMicrotask(() => {
      const batch = [...pendingUpdates];
      pendingUpdates.length = 0;
      
      db.batchUpdate(batch); // Single query
    });
  }
}

// Usage:
update({ id: 1, value: 'a' });
update({ id: 2, value: 'b' });
update({ id: 3, value: 'c' });
// → Send batch request khi tất cả microtasks done
```

#### Use Case 3: Animation Frame Scheduling

**RequestAnimationFrame**: Macrotask đặc biệt
- Chạy **trước** rendering
- ~60 FPS (16.67ms/frame)

**Pattern**:
```
Microtasks → requestAnimationFrame → Render → Macrotasks
```

```javascript
let frame = 0;

function animate() {
  // Update state (microtask may be queued here)
  Promise.resolve().then(() => {
    console.log('State update in microtask');
  });
  
  // Visual update
  element.style.transform = `translateX(${frame}px)`;
  frame++;
  
  // Schedule next frame (macrotask)
  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);
```

---

## 3. Promises - Internals & Production Patterns

### 3.1. Promise State Machine Internals

**Mô hình State Transition**: Unidirectional state machine với 3 states khả dĩ

```
                   ┌─────────┐
                   │ PENDING │  State ban đầu
                   └────┬────┘
                        │
           ┌────────────┴────────────┐
           │                         │
     resolve(value)            reject(reason)
           │                         │
           ▼                         ▼
    ┌──────────┐              ┌──────────┐
    │ FULFILLED│              │ REJECTED │  Terminal states
    └──────────┘              └──────────┘
```

**Đặc điểm của từng State**:
- **Pending**: State ban đầu; promise chưa fulfilled hoặc rejected
  - Internal `[[PromiseState]]`: "pending"
  - Chưa có `[[PromiseResult]]` value
  - Đang chờ resolution thông qua executor function

- **Fulfilled**: Operation hoàn thành thành công
  - State transition là **không thể đảo ngược** (irreversible)
  - `[[PromiseResult]]` chứa resolution value
  - Tất cả registered `.then(onFulfilled)` handlers enqueued as microtasks

- **Rejected**: Operation failed với error
  - State transition là **không thể đảo ngược** (irreversible)
  - `[[PromiseResult]]` chứa rejection reason
  - Tất cả registered `.catch(onRejected)` handlers enqueued as microtasks

**Thuộc tính Critical**:
1. **Immutability**: Sau khi settled (fulfilled/rejected), state và value không thể thay đổi
2. **Handler Registration**: Callbacks attach sau settlement execute immediately (as microtasks)
3. **Chain Creation**: Mỗi `.then()/.catch()/.finally()` return **Promise mới**, enabling composition

### 3.2. Promise Internal Structure (Simplified)

```javascript
// Simplified internal structure
class MyPromise {
  constructor(executor) {
    this.state = 'pending';
    this.value = undefined;
    this.reason = undefined;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];
    
    const resolve = (value) => {
      if (this.state === 'pending') {
        this.state = 'fulfilled';
        this.value = value;
        this.onFulfilledCallbacks.forEach(fn => fn(value));
      }
    };
    
    const reject = (reason) => {
      if (this.state === 'pending') {
        this.state = 'rejected';
        this.reason = reason;
        this.onRejectedCallbacks.forEach(fn => fn(reason));
      }
    };
    
    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }
  
  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      if (this.state === 'fulfilled') {
        queueMicrotask(() => {
          try {
            const result = onFulfilled(this.value);
            resolve(result);
          } catch (error) {
            reject(error);
          }
        });
      }
      
      if (this.state === 'rejected') {
        queueMicrotask(() => {
          try {
            const result = onRejected(this.reason);
            resolve(result);
          } catch (error) {
            reject(error);
          }
        });
      }
      
      if (this.state === 'pending') {
        this.onFulfilledCallbacks.push((value) => {
          queueMicrotask(() => {
            try {
              const result = onFulfilled(value);
              resolve(result);
            } catch (error) {
              reject(error);
            }
          });
        });
        
        this.onRejectedCallbacks.push((reason) => {
          queueMicrotask(() => {
            try {
              const result = onRejected(reason);
              resolve(result);
            } catch (error) {
              reject(error);
            }
          });
        });
      }
    });
  }
  
  catch(onRejected) {
    return this.then(null, onRejected);
  }
  
  finally(onFinally) {
    return this.then(
      value => {
        onFinally();
        return value;
      },
      reason => {
        onFinally();
        throw reason;
      }
    );
  }
}
```

### 3.3. Promise Chaining - Anti-Patterns và Best Practices

**Anti-Pattern 1: Nested Promises ("Callback Hell 2.0")**
```javascript
// ❌ Promise nesting - defeats the purpose
getUserData()
  .then(user => {
    return getUserPosts(user.id)
      .then(posts => {
        return getPostComments(posts[0].id)
          .then(comments => {
            // Deeply nested, hard to read
            return { user, posts, comments };
          });
      });
  });
```

**Best Practice: Flat Promise Chain**
```javascript
// ✅ Flat chain - readable and composable
let userData, postsData;

getUserData()
  .then(user => {
    userData = user;
    return getUserPosts(user.id);
  })
  .then(posts => {
    postsData = posts;
    return getPostComments(posts[0].id);
  })
  .then(comments => ({ 
    user: userData, 
    posts: postsData, 
    comments 
  }))
  .catch(error => {
    // Centralized error handling for entire chain
    console.error('Pipeline failed:', error);
  });
```

**Lợi thế chính**:
- **Flat structure**: Loại bỏ rightward drift, cải thiện readability
- **Error propagation**: Single `.catch()` xử lý errors từ bất kỳ step nào
- **Type inference**: Mỗi `.then()` cung cấp clear input/output contract

### 3.4. Promise Static Methods - Strategic Selection Guide

#### Promise.all() - Parallel Execution với Fail-Fast Semantics

**Use Cases tối ưu**:
- **Independent resource fetching**: Load user data, settings, và permissions đồng thời
- **Parallel computation**: Xử lý multiple data chunks simultaneously
- **Atomic operations**: All-or-nothing batch updates (database transactions)

**Đặc điểm Execution**:
```javascript
const results = await Promise.all([
  fetchUser(),        // 100ms
  fetchSettings(),    // 150ms  
  fetchPermissions()  // 120ms
]);
// Tổng thời gian: max(100, 150, 120) = 150ms (không phải 370ms sequential)
// Returns: [userData, settingsData, permissionsData]
```

**Giới hạn quan trọng**:
- **Fail-fast behavior**: Rejection đầu tiên immediately rejects toàn bộ Promise.all()
  ```javascript
  Promise.all([
    Promise.resolve(1),
    Promise.reject(new Error('fail')), // Rejection ở đây
    Promise.resolve(3),
  ]); // Immediately rejects, results [1, 3] bị mất
  ```
- **No partial results**: Successful promises' values không accessible khi rejection
- **Resource waste**: Các requests khác vẫn tiếp tục execute nhưng results bị discard

**Khi nào nên tránh**: Operations mà partial success là acceptable (dùng `Promise.allSettled()`)

**Implementation (Simplified)**:
```javascript
Promise.myAll = function(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let completedCount = 0;
    
    if (promises.length === 0) {
      resolve(results);
      return;
    }
    
    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completedCount++;
          
          if (completedCount === promises.length) {
            resolve(results);
          }
        })
        .catch(reject); // Reject ngay khi có 1 promise fail
    });
  });
};
```

#### Promise.race() - First Completion Wins

**Use Cases**:

**1. Timeout Pattern**:
```javascript
function fetchWithTimeout(url, timeout = 5000) {
  return Promise.race([
    fetch(url),
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Request timeout')), timeout)
    )
  ]);
}
```

**2. Fastest Server (Redundant Requests)**:
```javascript
// Race giữa multiple servers, dùng response nhanh nhất
const data = await Promise.race([
  fetch('https://api1.example.com/data'),
  fetch('https://api2.example.com/data'),
  fetch('https://api3.example.com/data')
]);
// → Tăng reliability và giảm latency
```

**3. User Action vs Auto-save**:
```javascript
// Race giữa user click save vs auto-save timeout
await Promise.race([
  userClickSave(),      // User triggers save
  autoSaveTimeout(30000) // Auto-save after 30s
]);
```

**Implementation (Simplified)**:
```javascript
Promise.myRace = function(promises) {
  return new Promise((resolve, reject) => {
    promises.forEach(promise => {
      Promise.resolve(promise)
        .then(resolve)  // Resolve ngay khi có promise đầu tiên hoàn thành
        .catch(reject); // Reject ngay khi có promise đầu tiên fail
    });
  });
};
```

#### Promise.allSettled() - Fault-Tolerant Aggregation Pattern

**Triết lý thiết kế**: Chờ tất cả promises bất kể outcome, không bao giờ reject

**Cấu trúc Return Value**:
```javascript
const results = await Promise.allSettled([
  Promise.resolve(42),
  Promise.reject(new Error('failed')),
  Promise.resolve('success'),
]);

// results structure:
[
  { status: 'fulfilled', value: 42 },
  { status: 'rejected', reason: Error('failed') },
  { status: 'fulfilled', value: 'success' }
]
```

**Production Use Case: Microservices Aggregation**
```javascript
// Scenario: Dashboard fetching từ 5 microservices
const serviceResults = await Promise.allSettled([
  fetchUserService(),       // Success: 200ms
  fetchAnalyticsService(),  // Timeout: 5000ms
  fetchNotificationService(), // Success: 150ms
  fetchPaymentService(),    // Error 500: 300ms
  fetchInventoryService(),  // Success: 180ms
]);

// Graceful degradation strategy
const successfulData = Object.fromEntries(
  serviceResults
    .map((result, idx) => [
      serviceNames[idx],
      result.status === 'fulfilled' ? result.value : null
    ])
    .filter(([_, value]) => value !== null)
);

// Dashboard hiển thị: User data (✓), Notifications (✓), Inventory (✓)
// Gracefully handles: Analytics timeout, Payment service error
// User thấy: Partial UI với error indicators cho failed services
```

**Lợi ích**:
- **Partial degradation**: Application vẫn functional dù có partial failures
- **Complete observability**: Biết chính xác operations nào succeeded/failed
- **Better UX**: Hiển thị available data thay vì blank error screen

**Implementation (Simplified)**:
```javascript
Promise.myAllSettled = function(promises) {
  return new Promise(resolve => {
    const results = [];
    let completedCount = 0;
    
    if (promises.length === 0) {
      resolve(results);
      return;
    }
    
    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = { status: 'fulfilled', value };
        })
        .catch(reason => {
          results[index] = { status: 'rejected', reason };
        })
        .finally(() => {
          completedCount++;
          if (completedCount === promises.length) {
            resolve(results);
          }
        });
    });
  });
};
```

#### Promise.any() - First Success Wins

**Use case**: Chờ promise đầu tiên succeed, ignore rejections cho đến khi tất cả fail

```javascript
// Example: Try multiple CDNs
const script = await Promise.any([
  loadScript('https://cdn1.example.com/lib.js'),
  loadScript('https://cdn2.example.com/lib.js'),
  loadScript('https://cdn3.example.com/lib.js')
]);
// → Load từ CDN nhanh nhất available
```

**Khi nào dùng**:
- Fallback mechanisms với multiple sources
- Load balancing across redundant servers
- First successful response sufficient

### 3.5. Unhandled Promise Rejections

**Vấn đề nghiêm trọng**: Promise rejections không được catch dẫn đến silent failures

**Node.js Global Handler**:
```javascript
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  
  // Log to monitoring service (Sentry, Datadog, etc.)
  logger.error('Unhandled Promise Rejection', {
    reason,
    stack: reason?.stack,
    timestamp: Date.now()
  });
  
  // Graceful shutdown nếu critical
  if (isCriticalError(reason)) {
    process.exit(1);
  }
});
```

**Browser Global Handler**:
```javascript
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled Promise Rejection:', event.reason);
  
  // Prevent default browser error handling
  event.preventDefault();
  
  // Log to error tracking service
  if (window.Sentry) {
    Sentry.captureException(event.reason);
  }
  
  // Hiển thị error UI cho user
  showErrorToast({
    message: 'Something went wrong. Please try again.',
    technical: event.reason.message
  });
});
```

**Best Practices**:
- **Luôn luôn** dùng `.catch()` hoặc `try-catch` với async/await
- Global handlers làm safety net, không phải primary error handling
- Monitor unhandled rejections trong production và set up alerts
- Log đầy đủ context (stack trace, user state, request details)

---

## 4. Async/Await - Advanced Techniques

### 4.1. Async/Await Là Gì Thực Sự?

**Bản chất**: Syntactic sugar trên Promises + Generators

**Cơ chế Internally**:
```javascript
async function fetchData() {  // → Returns Promise
  const result = await apiCall(); // → Pause execution, yield to Event Loop
  return result;  // → Code sau await được wrapped trong .then()
}
```

**Compiler transformation (conceptual)**:
```javascript
// Code bạn viết
async function example() {
  const a = await fetch('/api/a');
  const b = await fetch('/api/b');
  return { a, b };
}

// Tương đương với
function example() {
  return fetch('/api/a')
    .then(a => fetch('/api/b')
    .then(b => ({ a, b })));
}
```

**Lợi thế chính**:
- **Synchronous-looking code**: Dễ đọc và hiểu hơn Promise chains
- **Better error handling**: Dùng try-catch quen thuộc thay vì .catch()
- **Easier debugging**: Call stack được preserved, dễ debug hơn

### 4.2. Sequential vs Parallel Execution - Performance Critical

**Anti-pattern phổ biến**: Unintentional sequential execution

```javascript
// ❌ Await trong loop → xử lý tuần tự (chậm)
async function fetchAllUsers(userIds) {
  const users = [];
  for (const id of userIds) {
    const user = await fetchUser(id);  // Chờ mỗi request xong mới tiếp tục
    users.push(user);
  }
  return users;
}
// Tổng thời gian: 10 users × 100ms = 1000ms
```

**Best practice**: Parallel execution khi possible

```javascript
// ✅ Promise.all → xử lý song song (nhanh)
async function fetchAllUsers(userIds) {
  const promises = userIds.map(id => fetchUser(id));
  return await Promise.all(promises);
}
// Tổng thời gian: max(100ms) = 100ms (parallel)
// Performance improvement: 10x faster!
```

**When to use sequential** (ngoại lệ):
- Requests phụ thuộc nhau (result của request 1 cần cho request 2)
- Rate limiting considerations
- Resource constraints (memory, connections)
- Waterfall dependencies

```javascript
// Sequential là cần thiết khi có dependency
async function processUserData() {
  const user = await fetchUser();
  const posts = await fetchUserPosts(user.id); // Cần user.id
  const comments = await fetchPostComments(posts[0].id); // Cần posts
  return { user, posts, comments };
}
```

### 4.3. Error Handling Patterns - Chiến Lược Tối Ưu

#### Pattern 1: Try-Catch per Operation (Granular)

```javascript
async function processData() {
  let userData, settingsData;
  
  try {
    userData = await fetchUser();
  } catch (error) {
    console.error('User fetch failed:', error);
    userData = getDefaultUser(); // Fallback
  }
  
  try {
    settingsData = await fetchSettings();
  } catch (error) {
    console.error('Settings fetch failed:', error);
    settingsData = getDefaultSettings(); // Fallback
  }
  
  return { userData, settingsData };
}
```

**Pros**: Granular error handling, specific fallbacks
**Cons**: Verbose, nhiều boilerplate code

#### Pattern 2: Single Try-Catch for Entire Function (Concise)

```javascript
async function processData() {
  try {
    const userData = await fetchUser();
    const settingsData = await fetchSettings();
    return { userData, settingsData };
  } catch (error) {
    console.error('Process failed:', error);
    return getDefaultData();
  }
}
```

**Pros**: Concise, dễ đọc
**Cons**: Không differentiate được errors, tất cả lỗi đều xử lý giống nhau

#### Pattern 3: Error-First Approach (Go-style)

```javascript
async function fetchData() {
  const [error, data] = await fetchUser()
    .then(data => [null, data])
    .catch(error => [error, null]);
  
  if (error) {
    // Explicit error handling
    return handleError(error);
  }
  
  return processData(data);
}
```

**Pros**: Explicit, forces error handling
**Cons**: Nhiều boilerplate

**Recommendation**: Dùng Pattern 2 cho most cases, Pattern 1 khi cần specific error handling

### 4.4. Advanced Async Patterns

#### Pattern: Retry with Exponential Backoff

```javascript
async function fetchWithRetry(url, options = {}) {
  const {
    maxRetries = 3,
    retryDelay = 1000,
    backoff = 'exponential' // 'exponential' | 'linear'
  } = options;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fetch(url);
    } catch (error) {
      const isLastAttempt = i === maxRetries - 1;
      
      if (isLastAttempt) {
        throw error;
      }
      
      const delay = backoff === 'exponential'
        ? retryDelay * Math.pow(2, i)
        : retryDelay * (i + 1);
      
      console.log(`Retry ${i + 1}/${maxRetries} after ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
const data = await fetchWithRetry('/api/data', {
  maxRetries: 3,
  retryDelay: 1000,
  backoff: 'exponential'
});
```

#### Pattern: Timeout Wrapper

```javascript
function withTimeout(promise, timeoutMs) {
  return Promise.race([
    promise,
    new Promise((_, reject) => {
      setTimeout(() => {
        reject(new Error(`Timeout after ${timeoutMs}ms`));
      }, timeoutMs);
    })
  ]);
}

// Usage
try {
  const data = await withTimeout(
    fetch('/api/slow-endpoint'),
    5000 // 5s timeout
  );
} catch (error) {
  console.error('Request timeout or failed:', error);
}
```

#### Pattern: Promise Pool (Concurrency Limit)

```javascript
async function promisePool(tasks, concurrency) {
  const results = [];
  const executing = [];
  
  for (const task of tasks) {
    const promise = Promise.resolve().then(() => task());
    results.push(promise);
    
    if (concurrency <= tasks.length) {
      const executingPromise = promise.then(() => {
        executing.splice(executing.indexOf(executingPromise), 1);
      });
      
      executing.push(executingPromise);
      
      if (executing.length >= concurrency) {
        await Promise.race(executing);
      }
    }
  }
  
  return Promise.all(results);
}

// Usage: Fetch 100 URLs with max 5 concurrent requests
const urls = Array.from({ length: 100 }, (_, i) => `/api/item/${i}`);
const tasks = urls.map(url => () => fetch(url));

const results = await promisePool(tasks, 5);
```

---

## 5. Advanced Async Patterns

### 5.1. AbortController - Request Cancellation

**Problem**: Không thể cancel in-flight async operations

**Solution**: AbortController API (modern browsers & Node.js 15+)

**Basic Usage**:
```javascript
const controller = new AbortController();
const signal = controller.signal;

// Start fetch with abort signal
fetch('/api/data', { signal })
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => {
    if (error.name === 'AbortError') {
      console.log('Request cancelled');
    } else {
      console.error('Request failed:', error);
    }
  });

// Cancel request after 5s
setTimeout(() => controller.abort(), 5000);
```

**Use Case: Search Autocomplete**:
```javascript
let currentController = null;

async function searchAPI(query) {
  // Cancel previous request
  if (currentController) {
    currentController.abort();
  }
  
  // Create new controller
  currentController = new AbortController();
  
  try {
    const response = await fetch(`/api/search?q=${query}`, {
      signal: currentController.signal
    });
    return await response.json();
  } catch (error) {
    if (error.name === 'AbortError') {
      console.log('Search cancelled');
      return null;
    }
    throw error;
  }
}

// User types rapidly
searchAPI('h');   // Starts request
searchAPI('he');  // Cancels previous, starts new
searchAPI('hel'); // Cancels previous, starts new
// Only last request completes
```

**Khi nào dùng**:
- User-initiated requests (search, autocomplete)
- Navigating giữa pages
- Timeout implementations
- Component cleanup (React useEffect)

### 5.2. Debounce & Throttle - Rate Limiting

#### Debounce: Chờ user ngừng input

```javascript
function debounce(func, delay) {
  let timeoutId;
  
  return function(...args) {
    clearTimeout(timeoutId);
    
    timeoutId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}

// Use case: Search autocomplete
const searchDebounced = debounce(async (query) => {
  const results = await fetch(`/api/search?q=${query}`);
  displayResults(results);
}, 300);

// User types "hello"
// h -> start timer (300ms)
// he -> reset timer
// hel -> reset timer  
// hell -> reset timer
// hello -> reset timer
// (300ms là không gõ gì) -> API call!

searchInput.addEventListener('input', (e) => {
  searchDebounced(e.target.value);
});
```

#### Throttle: Giới hạn tần suất

```javascript
function throttle(func, limit) {
  let inThrottle;
  
  return function(...args) {
    if (!inThrottle) {
      func.apply(this, args);
      inThrottle = true;
      
      setTimeout(() => {
        inThrottle = false;
      }, limit);
    }
  };
}

// Use case: Scroll events
const handleScrollThrottled = throttle(() => {
  console.log('Scroll position:', window.scrollY);
  // Update UI based on scroll
}, 100);

window.addEventListener('scroll', handleScrollThrottled);
// Execute tối đa 1 lần/100ms
// Events giữa các lần execute bị discard
```

**Khi nào dùng**:
- **Debounce**: Search, form validation, window resize
- **Throttle**: Scroll events, mousemove, game loop updates

### 5.3. Race Conditions - Patterns & Solutions

#### Type 1: Request Race

**Problem**: Later request starts first, earlier finishes last → stale data

**Solution 1: Request ID Tracking**:
```javascript
let latestRequestId = 0;

async function loadUser(userId) {
  const requestId = ++latestRequestId;
  
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();
  
  // Chỉ update nếu đây là request mới nhất
  if (requestId === latestRequestId) {
    userData = data;
  }
}
```

**Solution 2: AbortController** (Preferred):
```javascript
let currentController = null;

async function loadUser(userId) {
  if (currentController) {
    currentController.abort();
  }
  
  currentController = new AbortController();
  
  const response = await fetch(`/api/users/${userId}`, {
    signal: currentController.signal
  });
  userData = await response.json();
}
```

#### Type 2: State Race

**Problem**: Multiple async updates to same state

```javascript
// ❌ BAD: Race condition
let count = 0;

async function increment() {
  const current = count;
  await delay(100);
  count = current + 1;
}

increment(); // count = 0 → 1
increment(); // count = 0 → 1 (WRONG! Should be 2)
```

**Solution: Atomic updates** (React functional updates):
```javascript
const [count, setCount] = useState(0);

function increment() {
  setCount(prev => prev + 1); // Always correct, based on latest state
}
```

#### Type 3: Database Race

**Problem**: Read-modify-write race condition

```javascript
// ❌ BAD: Non-atomic update
async function incrementLikes(postId) {
  const post = await db.posts.findOne({ id: postId });
  post.likes += 1;
  await db.posts.update({ id: postId }, post);
}
// Two concurrent calls can both read likes=10, both set to 11

// ✅ GOOD: Atomic update
async function incrementLikes(postId) {
  await db.posts.update(
    { id: postId },
    { $inc: { likes: 1 } } // Atomic increment
  );
}
```

### 5.4. Mutex Lock Pattern

**Use case**: Ensure sequential execution of critical sections

```javascript
class Mutex {
  constructor() {
    this.locked = false;
    this.queue = [];
  }
  
  async lock() {
    if (!this.locked) {
      this.locked = true;
      return;
    }
    
    return new Promise(resolve => {
      this.queue.push(resolve);
    });
  }
  
  unlock() {
    if (this.queue.length > 0) {
      const resolve = this.queue.shift();
      resolve();
    } else {
      this.locked = false;
    }
  }
}

// Usage
const mutex = new Mutex();

async function criticalSection() {
  await mutex.lock();
  
  try {
    // Critical code - only one execution at a time
    const data = await readData();
    await processData(data);
    await writeData(data);
  } finally {
    mutex.unlock();
  }
}
```

### 5.5. Circuit Breaker Pattern

**Purpose**: Prevent cascading failures, fast-fail when service is down

```javascript
class CircuitBreaker {
  constructor(fn, options = {}) {
    this.fn = fn;
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.nextAttempt = Date.now();
  }
  
  async execute(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await this.fn(...args);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failureCount++;
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
    }
  }
}

// Usage
const fetchWithCircuitBreaker = new CircuitBreaker(
  async (url) => {
    const response = await fetch(url);
    if (!response.ok) throw new Error('Fetch failed');
    return response.json();
  },
  {
    failureThreshold: 3,
    resetTimeout: 30000
  }
);

try {
  const data = await fetchWithCircuitBreaker.execute('/api/data');
} catch (error) {
  console.error('Circuit breaker prevented request:', error);
}
```

---

## 6. Async Iteration & Generators

### 6.1. Async Iterators - for await...of

**Concept**: Iterate over async data sources sequentially

**Async Iterable Protocol**:
```javascript
// Object phải implement Symbol.asyncIterator
const asyncIterable = {
  async *[Symbol.asyncIterator]() {
    yield 1;
    yield 2;
    yield 3;
  }
};

// Usage
for await (const value of asyncIterable) {
  console.log(value); // 1, 2, 3
}
```

**Use Case: Paginated API**:
```javascript
async function* fetchAllPages(url) {
  let page = 1;
  let hasMore = true;
  
  while (hasMore) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();
    
    yield data.items;  // Yield mỗi page
    
    hasMore = data.hasMore;
    page++;
  }
}

// Usage
for await (const items of fetchAllPages('/api/users')) {
  console.log('Processing page:', items.length, 'items');
  processItems(items);
}
```

**Lợi ích**:
- **Memory efficient**: Không load tất cả data vào memory cùng lúc
- **Lazy evaluation**: Chỉ fetch khi cần
- **Clean syntax**: For-await-of loop dễ đọc
- **Back-pressure support**: Có thể pause/resume iteration

### 6.2. Async Generators - async function*

**Syntax**: Combine async functions với generators

```javascript
async function* generateSequence() {
  for (let i = 1; i <= 3; i++) {
    await delay(1000); // Simulate async operation
    yield i;
  }
}

// Usage
const generator = generateSequence();

console.log(await generator.next()); // { value: 1, done: false } after 1s
console.log(await generator.next()); // { value: 2, done: false } after 2s
console.log(await generator.next()); // { value: 3, done: false } after 3s
console.log(await generator.next()); // { value: undefined, done: true }
```

**Use Case: Stream Processing**:
```javascript
async function* processLogFile(filePath) {
  const fileHandle = await fs.open(filePath, 'r');
  const stream = fileHandle.createReadStream();
  
  let buffer = '';
  
  for await (const chunk of stream) {
    buffer += chunk;
    const lines = buffer.split('\n');
    buffer = lines.pop(); // Keep incomplete line in buffer
    
    for (const line of lines) {
      const parsed = parseLogLine(line);
      if (parsed) {
        yield parsed;
      }
    }
  }
  
  // Process remaining buffer
  if (buffer.trim()) {
    const parsed = parseLogLine(buffer);
    if (parsed) yield parsed;
  }
  
  await fileHandle.close();
}

// Usage: Process large log file without loading all into memory
for await (const logEntry of processLogFile('/var/log/app.log')) {
  if (logEntry.level === 'ERROR') {
    console.error('Error found:', logEntry);
  }
}
```

**Use Case: Infinite Sequences**:
```javascript
async function* infiniteStream() {
  let id = 0;
  
  while (true) {
    // Fetch real-time data
    const data = await fetch('/api/stream');
    const json = await data.json();
    
    yield { id: id++, ...json };
    
    // Polling interval
    await delay(5000);
  }
}

// Usage with break condition
for await (const item of infiniteStream()) {
  processItem(item);
  
  if (shouldStop()) {
    break; // Exit infinite loop
  }
}
```

**Real-World Examples**:

**1. CSV Streaming**:
```javascript
async function* parseCSV(filePath) {
  const fileHandle = await fs.open(filePath);
  const stream = fileHandle.createReadStream();
  
  let buffer = '';
  let isFirstLine = true;
  let headers = [];
  
  for await (const chunk of stream) {
    buffer += chunk;
    const lines = buffer.split('\n');
    buffer = lines.pop();
    
    for (const line of lines) {
      if (isFirstLine) {
        headers = line.split(',');
        isFirstLine = false;
        continue;
      }
      
      const values = line.split(',');
      const row = Object.fromEntries(
        headers.map((h, i) => [h, values[i]])
      );
      
      yield row;
    }
  }
}

// Process 1GB CSV file
for await (const row of parseCSV('large-data.csv')) {
  await processRow(row); // Process one row at a time
}
```

**2. Database Cursor Streaming**:
```javascript
async function* queryStream(query) {
  const cursor = db.collection('users').find(query);
  
  while (await cursor.hasNext()) {
    yield await cursor.next();
  }
  
  await cursor.close();
}

// Process millions of records
for await (const user of queryStream({ active: true })) {
  await sendEmail(user);
}
```

### 6.3. Combining Async Iterators

**Pattern: Transform Stream**:
```javascript
async function* map(asyncIterable, transformFn) {
  for await (const item of asyncIterable) {
    yield await transformFn(item);
  }
}

async function* filter(asyncIterable, predicateFn) {
  for await (const item of asyncIterable) {
    if (await predicateFn(item)) {
      yield item;
    }
  }
}

// Usage: Pipeline
const numbers = fetchNumbers(); // async generator
const doubled = map(numbers, x => x * 2);
const evens = filter(doubled, x => x % 2 === 0);

for await (const num of evens) {
  console.log(num);
}
```

---

## 7. Error Handling in Production

### 7.1. Unhandled Promise Rejections

**Node.js Global Handler**:
```javascript
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise);
  console.error('Reason:', reason);
  
  // Log to monitoring service
  logger.error('Unhandled Promise Rejection', {
    reason,
    stack: reason?.stack,
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV
  });
  
  // Alert on-call engineer for critical errors
  if (isCriticalError(reason)) {
    alertOncall({
      severity: 'critical',
      message: 'Unhandled Promise Rejection',
      details: reason
    });
  }
});

// Graceful shutdown on critical errors
process.on('unhandledRejection', (reason) => {
  if (isCriticalError(reason)) {
    console.error('Critical error, shutting down...');
    
    // Close server gracefully
    server.close(() => {
      process.exit(1);
    });
    
    // Force exit after 10s if graceful shutdown fails
    setTimeout(() => {
      process.exit(1);
    }, 10000);
  }
});
```

**Browser Global Handler**:
```javascript
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled Promise Rejection:', event.reason);
  
  // Prevent default browser error handling
  event.preventDefault();
  
  // Log to error tracking service
  if (window.Sentry) {
    Sentry.captureException(event.reason, {
      tags: {
        type: 'unhandled_rejection'
      },
      extra: {
        promise: event.promise,
        timestamp: Date.now()
      }
    });
  }
  
  // Hiển thị user-friendly error
  showErrorToast({
    title: 'Something went wrong',
    message: 'We\'re working on fixing this issue.',
    technical: event.reason?.message,
    action: {
      label: 'Retry',
      onClick: () => window.location.reload()
    }
  });
});
```

### 7.2. Structured Error Handling

**Custom Error Classes**:
```javascript
class AsyncError extends Error {
  constructor(message, code, details) {
    super(message);
    this.name = 'AsyncError';
    this.code = code;
    this.details = details;
    this.timestamp = Date.now();
  }
}

class NetworkError extends AsyncError {
  constructor(message, url, status) {
    super(message, 'NETWORK_ERROR', { url, status });
    this.name = 'NetworkError';
  }
}

class ValidationError extends AsyncError {
  constructor(message, field, value) {
    super(message, 'VALIDATION_ERROR', { field, value });
    this.name = 'ValidationError';
  }
}

// Usage
async function fetchWithErrorHandling(url) {
  try {
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new NetworkError(
        'Fetch failed',
        url,
        response.status
      );
    }
    
    return await response.json();
  } catch (error) {
    if (error instanceof NetworkError) {
      // Handle network errors specifically
      logger.error('Network error', error.details);
      throw error;
    }
    
    // Wrap unknown errors
    throw new AsyncError(
      'Unknown error occurred',
      'UNKNOWN_ERROR',
      { url, originalError: error.message }
    );
  }
}
```

### 7.3. Error Boundaries for Async (React)

> **⚠️ Lưu ý**: React Error Boundaries **không** catch async errors theo mặc định. Cần wrapper pattern.

**Async Error Boundary Pattern**:
```javascript
// HOC wrapper cho async operations
function withAsyncErrorBoundary(asyncFn) {
  return async (...args) => {
    try {
      return await asyncFn(...args);
    } catch (error) {
      // Convert async error thành sync error để Error Boundary catch
      throw error;
    }
  };
}

// Usage trong component
function MyComponent() {
  const [data, setData] = useState(null);
  
  const fetchData = withAsyncErrorBoundary(async () => {
    const result = await fetch('/api/data');
    setData(result);
  });
  
  useEffect(() => {
    fetchData().catch(error => {
      // Re-throw để Error Boundary catch
      throw error;
    });
  }, []);
  
  return <div>{data}</div>;
}
```

**Better Pattern: Use Error State**:
```javascript
function MyComponent() {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    async function loadData() {
      try {
        const result = await fetch('/api/data');
        setData(result);
      } catch (err) {
        setError(err);
      }
    }
    
    loadData();
  }, []);
  
  if (error) {
    return <ErrorDisplay error={error} />;
  }
  
  return <div>{data}</div>;
}
```

### 7.4. Production Monitoring

**Error Tracking Services Integration**:

**Sentry**:
```javascript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});

// Capture async errors
async function monitoredFetch(url) {
  const transaction = Sentry.startTransaction({
    op: 'http',
    name: `GET ${url}`,
  });
  
  try {
    const response = await fetch(url);
    transaction.setStatus('ok');
    return response;
  } catch (error) {
    transaction.setStatus('internal_error');
    
    Sentry.captureException(error, {
      tags: { url },
      extra: { timestamp: Date.now() }
    });
    
    throw error;
  } finally {
    transaction.finish();
  }
}
```

**Performance Monitoring**:
```javascript
class PerformanceMonitor {
  static async measure(name, fn) {
    const start = performance.now();
    
    try {
      const result = await fn();
      const duration = performance.now() - start;
      
      console.log(`[${name}] Duration: ${duration.toFixed(2)}ms`);
      
      // Send to analytics
      this.sendMetric(name, duration, 'success');
      
      // Alert if slow
      if (duration > 5000) {
        alertSlowOperation(name, duration);
      }
      
      return result;
    } catch (error) {
      const duration = performance.now() - start;
      
      console.error(`[${name}] Failed after ${duration.toFixed(2)}ms`, error);
      
      this.sendMetric(name, duration, 'error');
      
      throw error;
    }
  }
  
  static sendMetric(name, duration, status) {
    // Send to monitoring service (Datadog, New Relic, etc.)
    if (window.gtag) {
      gtag('event', 'timing_complete', {
        name,
        value: Math.round(duration),
        event_category: 'async_operations',
        event_label: status
      });
    }
  }
}

// Usage
const data = await PerformanceMonitor.measure('fetchUserData', async () => {
  return await fetch('/api/user').then(r => r.json());
});
```

---

## 8. Performance Optimization

### 8.1. Identifying Bottlenecks

**Chrome DevTools Performance Profiler**:
1. Open DevTools → Performance tab
2. Record profile while using app
3. Analyze flame chart:
   - **Yellow bars**: Scripting (JavaScript execution)
   - **Purple bars**: Rendering (style calculation, layout)
   - **Green bars**: Painting
   - **Red indicators**: Long tasks (>50ms)

**Long Task API**:
```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn('Long task detected:', {
      duration: entry.duration,
      startTime: entry.startTime,
      name: entry.name
    });
    
    // Log to monitoring
    if (entry.duration > 100) {
      logLongTask({
        duration: entry.duration,
        timestamp: Date.now()
      });
    }
  }
});

observer.observe({ entryTypes: ['longtask'] });
```

### 8.2. Task Chunking

**Problem**: Long synchronous task blocks event loop

**Solution**: Split vào smaller chunks với macrotask scheduling

```javascript
// ❌ BAD: Blocks UI for entire duration
function processLargeArray(items) {
  const results = [];
  for (const item of items) {
    results.push(expensiveOperation(item));
  }
  return results;
}

// ✅ GOOD: Yields to event loop between chunks
async function processLargeArrayChunked(items, chunkSize = 1000) {
  const results = [];
  
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    
    for (const item of chunk) {
      results.push(expensiveOperation(item));
    }
    
    // Yield to event loop
    await new Promise(resolve => setTimeout(resolve, 0));
    
    // Update progress
    const progress = ((i + chunkSize) / items.length) * 100;
    updateProgressBar(Math.min(progress, 100));
  }
  
  return results;
}

// Usage
const items = Array.from({ length: 100000 }, (_, i) => i);
const results = await processLargeArrayChunked(items);
```

**Time-slicing with requestIdleCallback** (better cho non-critical work):
```javascript
async function processWhenIdle(items) {
  const results = [];
  let index = 0;
  
  while (index < items.length) {
    await new Promise(resolve => {
      requestIdleCallback((deadline) => {
        // Process while idle time available
        while (deadline.timeRemaining() > 0 && index < items.length) {
          results.push(expensiveOperation(items[index]));
          index++;
        }
        
        resolve();
      });
    });
  }
  
  return results;
}
```

### 8.3. Memoization cho Async Functions

```javascript
function memoizeAsync(fn, options = {}) {
  const cache = new Map();
  const { ttl = Infinity, maxSize = Infinity } = options;
  
  return async function(...args) {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      const { value, timestamp } = cache.get(key);
      
      // Check TTL
      if (Date.now() - timestamp < ttl) {
        return value;
      }
      
      cache.delete(key);
    }
    
    const value = await fn.apply(this, args);
    
    // Enforce max size (LRU)
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }
    
    cache.set(key, { 
      value, 
      timestamp: Date.now() 
    });
    
    return value;
  };
}

// Usage
const fetchUser = memoizeAsync(
  async (userId) => {
    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  },
  { 
    ttl: 60000,  // Cache for 1 minute
    maxSize: 100 // Max 100 entries
  }
);

await fetchUser(1); // Fetch from API
await fetchUser(1); // Return from cache (< 1 minute)
```

### 8.4. Lazy Loading

**Code Splitting với Dynamic Imports**:
```javascript
// Load heavy library only when needed
async function processImage(image) {
  // Load library dynamically
  const { default: imageProcessor } = await import('./heavy-image-lib.js');
  
  return imageProcessor.process(image);
}

// React lazy loading
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

**Image Lazy Loading**:
```javascript
// Intersection Observer for lazy loading
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src; // Load actual image
      imageObserver.unobserve(img);
    }
  });
});

document.querySelectorAll('img[data-src]').forEach(img => {
  imageObserver.observe(img);
});
```

### 8.5. Web Worker Patterns

**Worker Pool**:
```javascript
class WorkerPool {
  constructor(workerScript, poolSize = navigator.hardwareConcurrency || 4) {
    this.workers = [];
    this.queue = [];
    
    // Create worker pool
    for (let i = 0; i < poolSize; i++) {
      const worker = new Worker(workerScript);
      this.workers.push({ worker, busy: false });
    }
  }
  
  async execute(data) {
    return new Promise((resolve, reject) => {
      const task = { data, resolve, reject };
      
      // Try to assign to free worker
      const freeWorker = this.workers.find(w => !w.busy);
      
      if (freeWorker) {
        this.runTask(freeWorker, task);
      } else {
        this.queue.push(task);
      }
    });
  }
  
  runTask(workerObj, task) {
    workerObj.busy = true;
    
    const handleMessage = (e) => {
      workerObj.worker.removeEventListener('message', handleMessage);
      workerObj.worker.removeEventListener('error', handleError);
      workerObj.busy = false;
      
      task.resolve(e.data);
      
      // Process next queued task
      if (this.queue.length > 0) {
        const nextTask = this.queue.shift();
        this.runTask(workerObj, nextTask);
      }
    };
    
    const handleError = (error) => {
      workerObj.worker.removeEventListener('message', handleMessage);
      workerObj.worker.removeEventListener('error', handleError);
      workerObj.busy = false;
      
      task.reject(error);
    };
    
    workerObj.worker.addEventListener('message', handleMessage);
    workerObj.worker.addEventListener('error', handleError);
    workerObj.worker.postMessage(task.data);
  }
  
  terminate() {
    this.workers.forEach(({ worker }) => worker.terminate());
  }
}

// Usage
const pool = new WorkerPool('processor.js', 4);

// Process 100 items across 4 workers
const results = await Promise.all(
  items.map(item => pool.execute(item))
);
```

### 8.6. Core Web Vitals Monitoring

```javascript
// Largest Contentful Paint (LCP)
new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];
  
  console.log('LCP:', lastEntry.renderTime || lastEntry.loadTime);
  
  // Send to analytics
  sendMetric('LCP', lastEntry.renderTime || lastEntry.loadTime);
}).observe({ entryTypes: ['largest-contentful-paint'] });

// First Input Delay (FID)
new PerformanceObserver((list) => {
  const entries = list.getEntries();
  
  entries.forEach(entry => {
    console.log('FID:', entry.processingStart - entry.startTime);
    sendMetric('FID', entry.processingStart - entry.startTime);
  });
}).observe({ entryTypes: ['first-input'] });

// Cumulative Layout Shift (CLS)
let clsScore = 0;

new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (!entry.hadRecentInput) {
      clsScore += entry.value;
    }
  }
  
  console.log('CLS:', clsScore);
  sendMetric('CLS', clsScore);
}).observe({ entryTypes: ['layout-shift'] });
```

---

## 9. Testing Async Code

### 9.1. Testing Strategies

**Jest/Vitest với async/await**:
```javascript
describe('fetchUser', () => {
  it('should fetch user data successfully', async () => {
    const user = await fetchUser(1);
    
    expect(user).toEqual({
      id: 1,
      name: 'John Doe'
    });
  });
  
  it('should handle errors', async () => {
    await expect(fetchUser(999)).rejects.toThrow('User not found');
  });
});
```

**Testing với done() callback** (legacy):
```javascript
it('should complete async operation', (done) => {
  fetchUser(1).then(user => {
    expect(user.name).toBe('John Doe');
    done(); // Must call done()
  }).catch(done); // Pass done to catch for errors
});
```

### 9.2. Mocking Async Operations

**Mock Timers**:
```javascript
describe('debounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });
  
  afterEach(() => {
    jest.useRealTimers();
  });
  
  it('should debounce function calls', () => {
    const fn = jest.fn();
    const debounced = debounce(fn, 1000);
    
    debounced();
    debounced();
    debounced();
    
    expect(fn).not.toHaveBeenCalled();
    
    // Fast-forward time
    jest.advanceTimersByTime(1000);
    
    expect(fn).toHaveBeenCalledTimes(1);
  });
});
```

**Mock fetch**:
```javascript
// Using jest-fetch-mock
import fetchMock from 'jest-fetch-mock';

fetchMock.enableMocks();

beforeEach(() => {
  fetch.resetMocks();
});

it('should fetch user data', async () => {
  fetch.mockResponseOnce(JSON.stringify({ id: 1, name: 'John' }));
  
  const user = await fetchUser(1);
  
  expect(fetch).toHaveBeenCalledWith('/api/users/1');
  expect(user).toEqual({ id: 1, name: 'John' });
});

it('should handle fetch errors', async () => {
  fetch.mockReject(new Error('Network error'));
  
  await expect(fetchUser(1)).rejects.toThrow('Network error');
});
```

**Mock với MSW (Mock Service Worker)** - More realistic:
```javascript
import { rest } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    return res(
      ctx.json({ id: req.params.id, name: 'John Doe' })
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

it('should fetch user', async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe('John Doe');
});
```

### 9.3. Testing Race Conditions

```javascript
describe('race condition handling', () => {
  it('should only use latest request result', async () => {
    let currentController;
    const results = [];
    
    async function search(query) {
      if (currentController) {
        currentController.abort();
      }
      
      currentController = new AbortController();
      
      try {
        const response = await fetch(`/api/search?q=${query}`, {
          signal: currentController.signal
        });
        results.push(await response.json());
      } catch (error) {
        if (error.name !== 'AbortError') throw error;
      }
    }
    
    // Trigger multiple searches
    search('a');
    search('ab');
    await search('abc');
    
    // Only last search result should be present
    expect(results).toHaveLength(1);
    expect(results[0].query).toBe('abc');
  });
});
```

### 9.4. Testing Async Generators

```javascript
describe('async generator', () => {
  it('should yield all values', async () => {
    async function* generate() {
      yield 1;
      yield 2;
      yield 3;
    }
    
    const values = [];
    for await (const value of generate()) {
      values.push(value);
    }
    
    expect(values).toEqual([1, 2, 3]);
  });
  
  it('should handle errors in generator', async () => {
    async function* generateWithError() {
      yield 1;
      throw new Error('Generator error');
    }
    
    const values = [];
    
    await expect(async () => {
      for await (const value of