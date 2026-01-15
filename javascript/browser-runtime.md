# Browser Runtime & Web APIs

## 1. DOM Manipulation

### **Cấu trúc DOM Tree**

**Document Object Model** = Biểu diễn HTML dưới dạng cây (tree)

**Các loại Node:**
- Element nodes (thẻ HTML)
- Text nodes (nội dung text)
- Attribute nodes (deprecated)
- Comment nodes (comments)
- Document node (root)

---

### **Phương thức Selection**

**Modern (khuyên dùng):**
- `querySelector()` - Trả về element đầu tiên match (CSS selector)
- `querySelectorAll()` - NodeList (CSS selector)

**Legacy (tránh dùng trừ khi cần):**
- `getElementById()` - Single element
- `getElementsByClassName()` - Live HTMLCollection
- `getElementsByTagName()` - Live HTMLCollection

**Live vs Static Collections:**
- `getElementsBy*` → **Live** (tự động update khi DOM thay đổi)
- `querySelectorAll` → **Static** (snapshot tại thời điểm gọi)

---

### **Patterns Thao tác DOM**

**Tạo & Chèn:**
- `createElement()` + `appendChild()` / `insertBefore()`
- `insertAdjacentHTML()` - Parse string thành nodes
- `cloneNode(deep)` - Copy nodes

**Sửa:**
- `textContent` - Plain text (an toàn, chống XSS)
- `innerHTML` - HTML string (⚠️ XSS risk)
- `classList.add/remove/toggle` - Thao tác class
- `setAttribute()` / `removeAttribute()`

**Xóa:**
- `element.remove()` - Modern
- `parent.removeChild(child)` - Legacy

---

### **Performance Considerations**

**Reflow & Repaint:**
- **Reflow** (đắt): Thay đổi geometry (size, position)
- **Repaint** (rẻ hơn): Thay đổi visual (màu, visibility)

**Triggers:**
- Đọc layout properties buộc sync reflow (`offsetHeight`, `getBoundingClientRect()`)
- Batch DOM changes để giảm reflows

**Best Practices:**
- Dùng `DocumentFragment` cho nhiều insertions
- `requestAnimationFrame` cho animation updates
- Virtual DOM libraries (React) tránh thao tác trực tiếp

---

## 2. Events & Event Delegation

### **Event Flow: 3 Phases**

**1. Capture Phase** (root → target)
- Ít dùng, hữu ích để intercept events sớm
- `addEventListener(event, handler, true)` - capture

**2. Target Phase** (tại element target)

**3. Bubble Phase** (target → root)
- Mặc định, hầu hết events bubble
- `addEventListener(event, handler)` - bubble

**Exceptions không bubble:**
- `focus/blur` (dùng `focusin/focusout` thay thế)
- `load/unload`
- `mouseenter/mouseleave` (dùng `mouseover/mouseout`)

---

### **Event Delegation**

**Concept:** Attach listener lên parent, xử lý events từ children

**Lợi ích:**
- Ít listeners hơn → ít memory
- Dynamic elements tự động có handler
- Better performance với nhiều elements

**Pattern:**
```javascript
// Thay vì: 100 listeners cho 100 buttons
parent.addEventListener('click', (e) => {
  if (e.target.matches('.button-class')) {
    // Handle click
  }
});
```

**Khi nào dùng:**
- ✅ Danh sách items (todo list, tables)
- ✅ Dynamic content
- ❌ Unique handlers cho mỗi element
- ❌ Events không bubble

---

### **Event Object Methods**

**Kiểm soát flow:**
- `e.stopPropagation()` - Dừng bubble/capture
- `e.stopImmediatePropagation()` - Dừng cả listeners khác trên cùng element
- `e.preventDefault()` - Chặn default action (form submit, link navigate)

**Properties hữu ích:**
- `e.target` - Element trigger event
- `e.currentTarget` - Element có listener (this)
- `e.type` - Tên event ('click', 'input', etc.)
- `e.timeStamp` - Timestamp

---

### **Passive Events (Performance)**

**Problem:** Touch/wheel listeners block scrolling

**Solution:** Passive listeners
```javascript
// Báo browser: "Tôi sẽ không preventDefault()"
element.addEventListener('touchstart', handler, { passive: true });
```

**Khi nào dùng:**
- ✅ Touch/wheel events không cần preventDefault
- ✅ Scroll listeners chỉ đọc
- ❌ Khi cần preventDefault (validation, custom behaviors)

---

## 3. Web Storage (localStorage, sessionStorage)

### **localStorage vs sessionStorage**

| Feature | localStorage | sessionStorage |
|---------|-------------|----------------|
| **Lifetime** | Permanent (đến khi xóa) | Tab/window session |
| **Scope** | Same origin | Same origin + tab |
| **Capacity** | ~5-10MB | ~5-10MB |
| **Shared** | Across tabs | Isolated per tab |

---

### **API (Giống nhau cho cả 2)**

```javascript
// Set
localStorage.setItem('key', 'value');

// Get
const value = localStorage.getItem('key'); // null nếu không tồn tại

// Remove
localStorage.removeItem('key');

// Clear all
localStorage.clear();

// Check key exists
const exists = localStorage.getItem('key') !== null;
```

---

### **Lưu Objects (JSON Serialization)**

**Web Storage chỉ lưu strings:**
```javascript
// ❌ Không work
localStorage.setItem('user', { name: 'Alice' }); // "[object Object]"

// ✅ Phải serialize
const user = { name: 'Alice', age: 30 };
localStorage.setItem('user', JSON.stringify(user));

// ✅ Deserialize khi đọc
const retrieved = JSON.parse(localStorage.getItem('user'));
```

---

### **Limitations & Gotchas**

**Capacity:**
- Tràn quota → QuotaExceededError
- Không reliable cho data lớn

**Synchronous:**
- Block main thread (tránh trong hot paths)
- Ảnh hưởng performance nếu data lớn

**Security:**
- Same-origin policy
- Không encrypt (⚠️ đừng lưu sensitive data)
- XSS có thể đọc storage

**JSON limitations:**
- Mất functions, undefined, Dates (thành strings)
- Mất prototypes (objects thành plain objects)

---

### **Use Cases**

**localStorage:**
- User preferences (theme, language)
- Draft content (autosave)
- Cache data không sensitive

**sessionStorage:**
- Form data trong multi-step wizard
- Temporary state cho single session
- Tab-specific data

**Không dùng cho:**
- Sensitive data (tokens, passwords) → httpOnly cookies
- Large datasets → IndexedDB
- Cross-tab real-time sync → BroadcastChannel + storage events

---

### **Storage Events (Cross-tab Communication)**

**Trigger khi storage thay đổi từ tab khác:**
```javascript
window.addEventListener('storage', (e) => {
  // e.key - Key changed
  // e.oldValue - Previous value
  // e.newValue - New value
  // e.storageArea - localStorage hoặc sessionStorage
});
```

**Lưu ý:** Chỉ fire ở **tabs khác**, không fire ở tab trigger change

---

## 4. IndexedDB

### **Khi nào dùng IndexedDB**

**Dùng khi:**
- Cần lưu > 10MB data
- Structured data với indexes
- Offline-first apps
- Performance-critical reads/writes

**Không dùng khi:**
- Simple key-value (localStorage đủ)
- Small datasets
- Need server sync (consider cache strategies)

---

### **Key Concepts**

**Database → Object Stores → Records:**
- Database: Container (1 per app thường)
- Object Store: Như table trong SQL
- Record: Key-value pairs

**Indexes:**
- Query data by properties khác key
- Tạo index cho fields hay query

**Transactions:**
- Bắt buộc cho mọi operation
- Modes: `readonly`, `readwrite`
- Auto-commit hoặc explicit abort

**Asynchronous:**
- Tất cả operations đều async (callbacks/promises)
- Không block main thread

---

### **Async Nature & Patterns**

**Callback-based (native API):**
```javascript
const request = indexedDB.open('myDB', 1);
request.onsuccess = (e) => { /* ... */ };
request.onerror = (e) => { /* ... */ };
```

**Promise wrappers (khuyên dùng):**
- Dùng libraries: `idb`, `dexie.js`
- Cleaner async/await code
- Better error handling

---

### **Limitations**

**Complexity:**
- API verbose, khó dùng
- Setup requires schema versioning

**Storage limits:**
- Browser-dependent (~50% disk space available)
- User có thể clear
- Không persistent như server DB

**No server sync:**
- Chỉ local
- Cần custom sync logic nếu cần

**Cross-origin:**
- Same-origin policy
- Không share data giữa domains

---

### **Practical Pattern**

**Dùng wrapper library:**
- `idb` (Jake Archibald) - Minimal, Promise-based
- `Dexie.js` - Feature-rich, easier API
- Tránh dùng native API trừ khi cần tối ưu tuyệt đối

---

## 5. Web Workers

### **Concept**

**Web Workers** = JavaScript chạy trong background thread riêng

**Use cases:**
- Heavy computations (không block UI)
- Image/video processing
- Data parsing (large CSVs, JSON)
- Cryptography
- Background tasks

---

### **Communication: postMessage**

**Main thread → Worker:**
```javascript
worker.postMessage({ type: 'process', data: largeArray });
```

**Worker → Main thread:**
```javascript
self.postMessage({ type: 'result', value: computed });
```

**Message handler:**
```javascript
worker.onmessage = (e) => {
  console.log(e.data); // { type: 'result', value: ... }
};
```

---

### **Limitations**

**Không truy cập được:**
- ❌ DOM (document, window)
- ❌ Parent page variables
- ❌ localStorage, sessionStorage

**Có thể dùng:**
- ✅ XMLHttpRequest, fetch
- ✅ IndexedDB
- ✅ WebSockets
- ✅ Timers (setTimeout, setInterval)
- ✅ Pure computations

**Data transfer:**
- Structured Clone Algorithm (copy data)
- Large data = slow transfer
- Dùng Transferable Objects cho binary data (zero-copy)

---

### **Types of Workers**

**Dedicated Workers:**
- 1-to-1 với creating script
- Dùng `new Worker('worker.js')`

**Shared Workers:**
- Shared giữa multiple scripts/tabs
- Same origin
- Ít browser support hơn

**Service Workers:** (Xem section riêng)

---

### **When to Use**

**✅ Dùng khi:**
- Computation > 50ms (block UI)
- CPU-intensive tasks
- Background processing

**❌ Không cần khi:**
- Simple calculations
- Tasks cần DOM access
- Small datasets

---

## 6. Service Workers

### **Concept**

**Service Worker** = Proxy giữa web app và network

**Core features:**
- Intercept network requests
- Cache responses (offline support)
- Background sync
- Push notifications
- Event-driven (không chạy liên tục)

---

### **Lifecycle**

**1. Registration:**
```javascript
navigator.serviceWorker.register('/sw.js');
```

**2. Installation:**
- Trigger khi SW file thay đổi
- Cache static assets

**3. Activation:**
- Cleanup old caches
- Claim clients

**4. Idle:**
- Terminate khi không dùng
- Wake up khi có events

---

### **Events**

**install:**
- Cache static assets
- Chỉ fire 1 lần cho mỗi SW version

**activate:**
- Cleanup (xóa old caches)
- Take control của pages

**fetch:**
- Intercept every network request
- Return từ cache hoặc network

**sync:** (Background Sync API)
- Retry failed requests khi online

**push:** (Push API)
- Receive push notifications

---

### **Caching Strategies**

**Cache First:**
- Dùng cache nếu có, fallback network
- Tốt cho: Static assets

**Network First:**
- Try network trước, fallback cache
- Tốt cho: Dynamic content

**Cache Only:**
- Chỉ dùng cache
- Tốt cho: Offline-first apps

**Network Only:**
- Không cache
- Tốt cho: Real-time data

**Stale While Revalidate:**
- Return cache, update in background
- Tốt cho: Balance freshness & speed

---

### **Security Requirements**

**HTTPS mandatory:**
- Service Workers require HTTPS (except localhost)
- Security: Can intercept/modify requests

**Scope:**
- SW control pages trong directory của nó
- Đặt ở root để control toàn site

---

### **Debugging**

**Chrome DevTools:**
- Application tab → Service Workers
- View registered SW, status
- Unregister, update, stop

**Common issues:**
- Caching aggressive → Hard refresh bypass SW
- Update SW → May need wait/skip waiting

---

### **When to Use**

**✅ Dùng cho:**
- Progressive Web Apps (PWA)
- Offline functionality
- Background sync
- Push notifications

**❌ Không cần khi:**
- Simple static sites
- No offline requirements
- Server-rendered apps không cần offline

---

## 7. Fetch API

### **Fetch vs XMLHttpRequest**

**Fetch advantages:**
- Promise-based (cleaner async)
- Cleaner API
- Streaming responses
- Better CORS handling

**XMLHttpRequest advantages:**
- Upload progress events
- Synchronous requests (đừng dùng!)
- Older browser support

**Verdict:** Dùng Fetch cho mọi modern apps

---

### **Fetch Gotchas**

**1. Không reject HTTP errors:**
```javascript
// ❌ Này KHÔNG throw error cho 404, 500
fetch('/api/users').then(res => res.json());

// ✅ Phải check ok
fetch('/api/users')
  .then(res => {
    if (!res.ok) throw new Error('HTTP error');
    return res.json();
  });
```

**2. Credentials không gửi default:**
```javascript
// ❌ Cookies không gửi cho same-origin
fetch('/api/users');

// ✅ Include credentials
fetch('/api/users', { credentials: 'include' });
```

**3. Không abort được (nếu không dùng AbortController):**
- Request vẫn chạy ngay cả khi component unmount
- Memory leaks, race conditions

---

### **AbortController (Cancellation)**

**Pattern:**
```javascript
const controller = new AbortController();

fetch('/api/users', { signal: controller.signal })
  .then(res => res.json())
  .catch(err => {
    if (err.name === 'AbortError') {
      // Request cancelled
    }
  });

// Cancel request
controller.abort();
```

**Use cases:**
- Component unmount (React useEffect cleanup)
- User cancel (search debounce)
- Timeout

---

### **Request Options**

**Common options:**
```javascript
fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer token'
  },
  body: JSON.stringify(data),
  credentials: 'include', // Send cookies
  signal: abortSignal,
  cache: 'no-cache',
  mode: 'cors'
});
```

---

### **Response Methods**

**Parse body:**
- `response.json()` - Parse JSON
- `response.text()` - Plain text
- `response.blob()` - Binary (files, images)
- `response.arrayBuffer()` - Raw binary
- `response.formData()` - Form data

**Mỗi method chỉ gọi được 1 lần** (body stream consumed)

---

### **Error Handling**

**Network errors:** (DNS failure, offline)
- Fetch rejects promise

**HTTP errors:** (404, 500)
- Fetch **không reject**, phải check `response.ok`

**Pattern:**
```javascript
try {
  const response = await fetch('/api/users');
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }
  const data = await response.json();
} catch (error) {
  if (error.name === 'AbortError') {
    // Cancelled
  } else if (error instanceof TypeError) {
    // Network error
  } else {
    // HTTP error hoặc JSON parse error
  }
}
```

---

## 8. WebSockets

### **Concept**

**WebSocket** = Full-duplex, persistent connection

**So sánh HTTP:**
- HTTP: Request/response, stateless
- WebSocket: Bidirectional, persistent, real-time

---

### **Use Cases**

**✅ Khi nào dùng:**
- Real-time updates (chat, notifications)
- Live data (stock prices, sports scores)
- Multiplayer games
- Collaborative editing

**❌ Khi nào không dùng:**
- Simple request/response
- Infrequent updates (polling/long-polling đủ)
- File uploads (HTTP better)

---

### **Connection Lifecycle**

**1. Handshake (HTTP Upgrade):**
- Client gửi HTTP request với `Upgrade: websocket`
- Server accept → 101 Switching Protocols

**2. Open:**
- Connection established
- Có thể send/receive messages

**3. Message exchange:**
- Bidirectional, real-time

**4. Close:**
- Either side có thể close
- Cleanup resources

---

### **Events**

**open:**
- Connection established
- Ready to send messages

**message:**
- Receive data từ server
- `event.data` chứa message

**error:**
- Connection error
- Không có error details (security)

**close:**
- Connection closed
- `event.code`, `event.reason` cho details

---

### **Message Types**

**Text:**
```javascript
ws.send('Hello');
ws.send(JSON.stringify({ type: 'message', text: 'Hello' }));
```

**Binary:**
```javascript
ws.send(new Uint8Array([1, 2, 3]));
ws.send(blob);
```

**Receive:**
```javascript
ws.onmessage = (e) => {
  if (typeof e.data === 'string') {
    const message = JSON.parse(e.data);
  } else {
    // Binary data (Blob or ArrayBuffer)
  }
};
```

---

### **Reconnection Strategy**

**WebSocket không auto-reconnect:**

**Pattern: Exponential backoff**
```javascript
let reconnectDelay = 1000; // Start 1s
const maxDelay = 30000; // Max 30s

function connect() {
  const ws = new WebSocket('wss://example.com');
  
  ws.onclose = () => {
    setTimeout(() => {
      reconnectDelay = Math.min(reconnectDelay * 2, maxDelay);
      connect();
    }, reconnectDelay);
  };
  
  ws.onopen = () => {
    reconnectDelay = 1000; // Reset on success
  };
}
```

---

### **Security**

**Dùng WSS (WebSocket Secure):**
- `wss://` (như HTTPS)
- Encrypt traffic
- Avoid MITM attacks

**Authentication:**
- WebSocket không có headers
- Auth qua: URL query params, first message, hoặc HTTP cookies

**Validation:**
- Server phải validate origin
- Prevent unauthorized connections

---

### **Libraries (Khuyên dùng)**

**Socket.io:**
- Auto-reconnection
- Fallback (long-polling nếu WS fail)
- Room/namespace support
- Acknowledgments

**Why not native WebSocket:**
- Phải tự handle reconnection
- Không có fallback
- Verbose API

---

## 9. Intersection Observer

### **Concept**

**Intersection Observer** = Async observe khi element vào/ra viewport

**Thay thế:** Scroll listeners (better performance)

---

### **Use Cases**

**✅ Common uses:**
- Lazy loading images
- Infinite scroll
- Trigger animations khi vào view
- Track visibility (analytics)
- Defer non-critical resources

---

### **API**

**Tạo observer:**
```javascript
const observer = new IntersectionObserver(callback, options);
observer.observe(element);
```

**Callback:**
```javascript
const callback = (entries, observer) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      // Element vào viewport
      // Load image, trigger animation, etc.
      observer.unobserve(entry.target); // Stop observing
    }
  });
};
```

---

### **Options**

**root:**
- Viewport reference (default: browser viewport)
- Dùng element khác làm scroll container

**rootMargin:**
- Margin xung quanh root (như CSS margin)
- `"50px"` - Trigger 50px trước khi vào viewport
- `"-50px"` - Trigger 50px sau khi vào viewport

**threshold:**
- % visibility để trigger callback
- `0` - Bất kỳ pixel nào visible
- `1` - 100% visible
- `[0, 0.5, 1]` - Trigger at 0%, 50%, 100%

---

### **Entry Properties**

**entry.isIntersecting:**
- `true` nếu element intersecting với root

**entry.intersectionRatio:**
- % element visible (0 to 1)

**entry.boundingClientRect:**
- Element's bounding box

**entry.intersectionRect:**
- Phần visible của element

**entry.target:**
- Element đang observe

---

### **Pattern: Lazy Load Images**

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src; // Load image
      observer.unobserve(img); // Stop observing
    }
  });
});

document.querySelectorAll('img[data-src]').forEach(img => {
  observer.observe(img);
});
```

---

### **Performance Benefits**

**So với scroll listeners:**
- ✅ Không block main thread (async)
- ✅ Không cần throttle/debounce
- ✅ Browser-optimized
- ✅ Better battery life (mobile)

**When not to use:**
- Pixel-perfect scroll positions (dùng scroll events)
- Immediate feedback needed (scroll events)

---

## 10. Performance APIs

### **Navigation Timing**

**Đo page load performance:**

**Key metrics:**
- `domContentLoadedEventEnd - fetchStart` → DOM ready time
- `loadEventEnd - fetchStart` → Full page load time
- `responseEnd - requestStart` → Network time
- `domInteractive - domLoading` → DOM parse time

**Access:** `performance.timing` (deprecated) hoặc `performance.getEntriesByType('navigation')`

---

### **Resource Timing**

**Đo individual resource loads** (scripts, images, CSS):

**Metrics per resource:**
- Duration
- Transfer size
- Protocol
- Initiator type

**Access:** `performance.getEntriesByType('resource')`

---

### **User Timing (Custom Marks)**

**Tạo custom performance markers:**

```javascript
// Start mark
performance.mark('task-start');

// ... do work ...

// End mark
performance.mark('task-end');

// Measure
performance.measure('task-duration', 'task-start', 'task-end');

// Get measures
const measures = performance.getEntriesByName('task-duration');
console.log(measures[0].duration); // milliseconds
```

**Use cases:**
- Component render time
- API call duration
- Feature initialization time

---

### **Core Web Vitals**

**3 metrics Google quan trọng:**

**LCP (Largest Contentful Paint):**
- Time largest element painted
- Target: < 2.5s
- Measure user-perceived load speed

**FID (First Input Delay):**
- Time từ user input → browser respond
- Target: < 100ms
- Measure interactivity

**CLS (Cumulative Layout Shift):**
- Visual stability (elements shifting)
- Target: < 0.1
- Measure visual stability

**Đo bằng:** Web Vitals library hoặc Performance Observer

---

### **Performance Observer**

**Subscribe to performance entries:**

```javascript
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    console.log(entry.name, entry.duration);
  });
});

observer.observe({ entryTypes: ['measure', 'navigation', 'resource'] });
```

**Advantages:**
- Real-time notifications
- Không miss entries
- Efficient (không poll)

---

### **Memory Monitoring**

**Check memory usage:**
```javascript
if (performance.memory) {
  console.log(performance.memory.usedJSHeapSize);
  console.log(performance.memory.jsHeapSizeLimit);
}
```

**⚠️ Non-standard:** Chỉ Chrome

---

### **requestAnimationFrame (Rendering)**

**Schedule animation work:**

**Concept:**
- Callback trước next repaint
- ~60fps (16.67ms per frame)
- Pause khi tab hidden (battery-friendly)

**Pattern:**
```javascript
function animate() {
  // Update state
  element.style.transform = `translateX(${x}px)`;
  
  requestAnimationFrame(animate); // Loop
}

requestAnimationFrame(animate); // Start
```

**Use cases:**
- Smooth animations
- Canvas/WebGL rendering
- Scroll-linked effects

**Không dùng cho:**
- Non-visual work (dùng setTimeout/Web Workers)

---

### **requestIdleCallback (Background Work)**

**Schedule low-priority work:**

**Concept:**
- Callback khi browser idle (không có urgent work)
- Không block rendering/input

**Pattern:**
```javascript
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    const task = tasks.shift();
    processTask(task);
  }
  
  if (tasks.length > 0) {
    requestIdleCallback(callback); // Continue later
  }
});
```

**Use cases:**
- Analytics
- Background data processing
- Preloading resources
- Non-urgent DOM updates

**⚠️ Không dùng cho:**
- Time-critical tasks
- User-facing updates

---

### **Best Practices**

**Measuring:**
- ✅ Đo real user metrics (RUM)
- ✅ Đo trong production, not just dev
- ✅ Track trends over time
- ✅ Use Performance Observer cho real-time

**Optimizing:**
- ✅ Focus on Core Web Vitals
- ✅ Lazy load non-critical resources
- ✅ Code splitting
- ✅ Optimize images (WebP, lazy load)
- ✅ Minimize JavaScript execution time

**Monitoring:**
- ✅ Use browser DevTools
- ✅ Lighthouse CI
- ✅ Real User Monitoring tools (Sentry, DataDog)

---

## Quick Reference

### **Khi nào dùng gì:**

| Need | Use |
|------|-----|
| Simple key-value | localStorage |
| Large structured data | IndexedDB |
| Heavy computation | Web Workers |
| Offline support | Service Workers |
| Real-time bidirectional | WebSockets |
| Lazy load / infinite scroll | Intersection Observer |
| Smooth animations | requestAnimationFrame |
| Background tasks | requestIdleCallback |
| HTTP requests | Fetch API |
| Custom performance tracking | User Timing API |