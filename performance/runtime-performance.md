# PHẦN 3 — RUNTIME PERFORMANCE
Tối ưu code chạy trên browser, sau khi đã load xong

## 3.1 Runtime Performance – Hiểu bản chất trước khi optimize

### 📖 Main Thread – Bottleneck Lớn Nhất

**Concept cốt lõi:**
Browser có **1 main thread duy nhất** để:
- Execute JavaScript
- Calculate layout
- Paint pixels
- Handle user input

**Vấn đề:**
```
User click button
  ↓
Main thread đang bận (Long Task 500ms)
  ↓
Event handler phải đợi
  ↓
User thấy "app đơ"
```

👉 **Runtime optimization = Keep main thread responsive**

---

### 🎯 Frame Budget – 16.67ms Rule

**Target: 60 FPS (frames per second)**

```
1 second / 60 frames = 16.67ms per frame
```

**Trong 16.67ms, browser phải:**
1. Run JavaScript (~3-4ms)
2. Calculate layout (~2-3ms)
3. Paint (~2-3ms)
4. Composite (~1-2ms)
5. Browser overhead (~5ms)

**Reality check:**
```
Có ~10ms cho JavaScript execution mỗi frame

Nếu JS chạy > 10ms:
→ Frame dropped
→ Jank / stuttering
→ UX tệ
```

---

### 💡 Mental Model: The Pipeline

```
JavaScript → Style → Layout → Paint → Composite
    ↓          ↓        ↓        ↓         ↓
  Changes   Calc CSS  Geometry  Pixels   Layers
```

**Tối ưu từng stage:**
1. **JavaScript:** Reduce work, defer, Web Workers
2. **Style:** Reduce selectors complexity
3. **Layout:** Avoid layout thrashing
4. **Paint:** Reduce paint areas, use will-change
5. **Composite:** Use transform/opacity, GPU layers

---

## 3.2 JavaScript Execution Optimization

### 🎯 Debouncing – Giảm tần suất execution

**Problem:**
```javascript
// ❌ Event fire quá nhiều
input.addEventListener('input', (e) => {
  expensiveSearch(e.target.value); // Run mỗi keystroke
});

// User type "hello" (5 characters)
// → expensiveSearch() chạy 5 lần
```

**Solution: Debounce**
```javascript
function debounce(func, wait) {
  let timeout;
  return function executedFunction(...args) {
    clearTimeout(timeout);
    timeout = setTimeout(() => {
      func(...args);
    }, wait);
  };
}

// ✅ Debounced
input.addEventListener('input', debounce((e) => {
  expensiveSearch(e.target.value);
}, 300)); // Chỉ run sau 300ms idle

// User type "hello" → expensiveSearch() chỉ chạy 1 lần
```

**React Hook:**
```javascript
import { useState, useCallback } from 'react';

function useDebounce(callback, delay) {
  const [timeoutId, setTimeoutId] = useState(null);

  const debouncedCallback = useCallback((...args) => {
    if (timeoutId) clearTimeout(timeoutId);
    
    const id = setTimeout(() => {
      callback(...args);
    }, delay);
    
    setTimeoutId(id);
  }, [callback, delay, timeoutId]);

  return debouncedCallback;
}

// Usage
function SearchInput() {
  const [query, setQuery] = useState('');
  
  const handleSearch = useDebounce((value) => {
    performSearch(value);
  }, 300);

  return (
    <input 
      value={query}
      onChange={(e) => {
        setQuery(e.target.value);
        handleSearch(e.target.value);
      }}
    />
  );
}
```

**Khi nào dùng:**
- Search input
- Resize handlers
- Scroll handlers (sometimes)
- Auto-save functionality

---

### 🎯 Throttling – Giới hạn execution rate

**Problem:**
```javascript
// ❌ Scroll event fire hàng trăm lần/giây
window.addEventListener('scroll', () => {
  updateProgressBar(); // Run liên tục
});

// Scroll 1s → updateProgressBar() chạy 100+ lần
```

**Solution: Throttle**
```javascript
function throttle(func, limit) {
  let inThrottle;
  return function(...args) {
    if (!inThrottle) {
      func.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// ✅ Throttled
window.addEventListener('scroll', throttle(() => {
  updateProgressBar();
}, 100)); // Run tối đa 1 lần / 100ms

// Scroll 1s → updateProgressBar() chỉ chạy ~10 lần
```

**React Hook:**
```javascript
import { useRef, useCallback } from 'react';

function useThrottle(callback, delay) {
  const lastRan = useRef(Date.now());

  return useCallback((...args) => {
    const now = Date.now();
    
    if (now - lastRan.current >= delay) {
      callback(...args);
      lastRan.current = now;
    }
  }, [callback, delay]);
}

// Usage
function InfiniteScroll() {
  const handleScroll = useThrottle(() => {
    loadMoreItems();
  }, 200);

  useEffect(() => {
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);
}
```

**Khi nào dùng:**
- Scroll listeners
- Mouse move tracking
- Window resize
- Animation loops

---

### 📊 Debounce vs Throttle – Khi nào dùng cái nào?

| Use case | Debounce | Throttle | Why |
|----------|----------|----------|-----|
| **Search input** | ✅ | ❌ | Chỉ cần result cuối cùng |
| **Auto-save** | ✅ | ❌ | Save sau khi user ngừng typing |
| **Scroll progress** | ❌ | ✅ | Cần updates liên tục nhưng controlled |
| **Window resize** | ✅ | ❌ | Chỉ care final size |
| **Infinite scroll** | ❌ | ✅ | Load more khi scroll, nhưng không quá nhiều |
| **Mouse tracking** | ❌ | ✅ | Smooth updates, không skip |

---

### 🎯 RequestAnimationFrame – Sync với browser paint

**Problem:**
```javascript
// ❌ setTimeout/setInterval không sync với browser refresh
setInterval(() => {
  element.style.left = position + 'px';
  position += 5;
}, 16); // Cố sync với 60fps

// → Có thể miss frames hoặc double-paint
```

**Solution: requestAnimationFrame**
```javascript
// ✅ Sync với browser paint cycle
function animate() {
  element.style.left = position + 'px';
  position += 5;
  
  if (position < maxPosition) {
    requestAnimationFrame(animate);
  }
}

requestAnimationFrame(animate);
```

**Benefits:**
1. **Sync với refresh rate:** Browser tự schedule optimal timing
2. **Auto-pause:** Không chạy khi tab inactive (battery saving)
3. **Better performance:** Browser có thể optimize

---

**Pattern: Smooth scroll to element**
```javascript
function smoothScrollTo(element, duration = 1000) {
  const start = window.pageYOffset;
  const target = element.getBoundingClientRect().top + start;
  const distance = target - start;
  const startTime = performance.now();

  function scroll(currentTime) {
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);
    
    // Easing function (ease-in-out)
    const easeProgress = progress < 0.5
      ? 2 * progress * progress
      : 1 - Math.pow(-2 * progress + 2, 2) / 2;
    
    window.scrollTo(0, start + distance * easeProgress);
    
    if (progress < 1) {
      requestAnimationFrame(scroll);
    }
  }

  requestAnimationFrame(scroll);
}
```

---

**Pattern: FPS meter**
```javascript
let lastTime = performance.now();
let frames = 0;

function measureFPS() {
  frames++;
  const currentTime = performance.now();
  
  if (currentTime >= lastTime + 1000) {
    const fps = Math.round((frames * 1000) / (currentTime - lastTime));
    console.log(`FPS: ${fps}`);
    
    frames = 0;
    lastTime = currentTime;
  }
  
  requestAnimationFrame(measureFPS);
}

requestAnimationFrame(measureFPS);
```

---

### 🎯 Web Workers – Offload Heavy Computation

**Concept:**
Chạy JavaScript trên **background thread riêng**, không block main thread.

**Khi nào dùng:**
- Heavy computation (encryption, parsing, image processing)
- Large data processing (filtering, sorting 10000+ items)
- Background tasks (analytics, logging)

---

**Example: Heavy data filtering**

**Main thread (app.js):**
```javascript
// ❌ Heavy work on main thread
function filterLargeDataset(data, query) {
  // 500ms → UI frozen
  return data.filter(item => 
    item.title.includes(query) ||
    item.description.includes(query) ||
    item.tags.some(tag => tag.includes(query))
  );
}

// ✅ Offload to Web Worker
const worker = new Worker('filter-worker.js');

worker.postMessage({ data, query });

worker.onmessage = (event) => {
  const filteredData = event.data;
  updateUI(filteredData);
};
```

**Worker thread (filter-worker.js):**
```javascript
self.onmessage = (event) => {
  const { data, query } = event.data;
  
  // Heavy work runs on background thread
  const filtered = data.filter(item => 
    item.title.includes(query) ||
    item.description.includes(query) ||
    item.tags.some(tag => tag.includes(query))
  );
  
  // Send result back to main thread
  self.postMessage(filtered);
};
```

---

**React Hook for Web Worker:**
```javascript
import { useEffect, useRef, useState } from 'react';

function useWebWorker(workerFunction) {
  const workerRef = useRef(null);
  const [result, setResult] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    // Create worker from function
    const blob = new Blob([`(${workerFunction.toString()})()`]);
    const url = URL.createObjectURL(blob);
    workerRef.current = new Worker(url);

    workerRef.current.onmessage = (e) => setResult(e.data);
    workerRef.current.onerror = (e) => setError(e.message);

    return () => {
      workerRef.current?.terminate();
      URL.revokeObjectURL(url);
    };
  }, [workerFunction]);

  const execute = (data) => {
    workerRef.current?.postMessage(data);
  };

  return { result, error, execute };
}

// Usage
function DataProcessor() {
  const { result, execute } = useWebWorker(function() {
    self.onmessage = (e) => {
      const processed = e.data.map(item => heavyProcess(item));
      self.postMessage(processed);
    };
  });

  const handleProcess = () => {
    execute(largeDataset);
  };

  return <div>{result && <Results data={result} />}</div>;
}
```

---

**⚠️ Web Worker Limitations:**
1. ❌ **Không access DOM** (no document, window)
2. ❌ **Không share memory** với main thread (data copied)
3. ❌ **Setup overhead** (~50-100ms to spawn worker)

**Rule of thumb:**
- Task < 50ms → main thread OK
- Task 50-200ms → consider deferring
- Task > 200ms → Web Worker

---

## 3.3 Layout Thrashing – Tránh Forced Synchronous Layout

### 🎯 Bản chất của Layout Thrashing

**Browser rendering pipeline:**
```
JavaScript → Style → Layout → Paint → Composite
```

**Problem: Read-Write-Read-Write pattern**
```javascript
// ❌ Force browser relayout nhiều lần
elements.forEach(el => {
  const height = el.offsetHeight; // READ (force layout)
  el.style.width = height + 'px';  // WRITE (invalidate layout)
  
  const width = el.offsetWidth;    // READ (force layout again!)
  el.style.marginTop = width + 'px'; // WRITE
});

// 100 elements → 200 layouts! (cực kỳ chậm)
```

**Layout properties (đọc → force layout):**
- `offsetWidth`, `offsetHeight`
- `clientWidth`, `clientHeight`
- `scrollWidth`, `scrollHeight`
- `getBoundingClientRect()`
- `getComputedStyle()`

---

### ✅ Solution: Batch Reads & Writes

```javascript
// ✅ Batch all reads, then all writes
const measurements = elements.map(el => ({
  height: el.offsetHeight,
  width: el.offsetWidth
})); // All READs first

elements.forEach((el, i) => {
  el.style.width = measurements[i].height + 'px';
  el.style.marginTop = measurements[i].width + 'px';
}); // All WRITEs after

// 100 elements → 2 layouts total ✅
```

---

### 📋 Real-world Pattern: Accordion

```javascript
// ❌ Layout thrashing
function toggleAccordion(element) {
  if (element.classList.contains('open')) {
    const height = element.scrollHeight; // READ
    element.style.height = height + 'px'; // WRITE
    element.classList.remove('open'); // WRITE
  }
}

// ✅ No layout thrashing
function toggleAccordion(element) {
  // Use CSS transition instead
  element.classList.toggle('open');
}

/* CSS */
.accordion {
  height: 0;
  overflow: hidden;
  transition: height 0.3s;
}

.accordion.open {
  height: auto; /* Browser handles this efficiently */
}
```

---

### 📋 Tool: FastDOM library

**Automatically batch reads/writes:**
```javascript
import fastdom from 'fastdom';

// ✅ FastDOM handles batching
elements.forEach(el => {
  fastdom.measure(() => {
    const height = el.offsetHeight;
    
    fastdom.mutate(() => {
      el.style.width = height + 'px';
    });
  });
});
```

---

## 3.4 CSS Performance – Render Pipeline Optimization

### 🎯 CSS Triggers – Biết property nào trigger gì

**3 categories:**

#### 1️⃣ Layout (Geometry) – Most expensive
Trigger: **Layout → Paint → Composite**

```css
/* ❌ Trigger layout */
width, height, margin, padding, border
top, left, right, bottom
font-size, line-height
```

**Impact:** ~5-10ms (desktop), 20-50ms (mobile)

---

#### 2️⃣ Paint (Visual) – Medium cost
Trigger: **Paint → Composite** (skip layout)

```css
/* ⚠️ Trigger paint */
color, background, box-shadow
border-radius, visibility
```

**Impact:** ~2-5ms

---

#### 3️⃣ Composite (Layers) – Cheapest
Trigger: **Composite only** (skip layout & paint)

```css
/* ✅ Cheap (GPU accelerated) */
transform, opacity
filter (với will-change)
```

**Impact:** <1ms

---

### ✅ Optimization Strategy: Use transform/opacity

```css
/* ❌ Trigger layout */
.box {
  left: 100px;
  top: 50px;
}

.box:hover {
  left: 200px; /* Expensive! */
}

/* ✅ Use transform (composite only) */
.box {
  transform: translate(100px, 50px);
}

.box:hover {
  transform: translate(200px, 50px); /* Cheap! */
}
```

---

**Animation performance:**
```css
/* ❌ Janky animation (60fps → 30fps) */
@keyframes slideIn {
  from { left: -100px; }
  to { left: 0; }
}

/* ✅ Smooth animation (60fps) */
@keyframes slideIn {
  from { transform: translateX(-100px); }
  to { transform: translateX(0); }
}
```

---

### 🎯 will-change – Hint Browser tạo layer

```css
/* Tạo GPU layer trước khi animate */
.modal {
  will-change: transform, opacity;
}

/* Sau khi animate xong, remove */
.modal.animated {
  will-change: auto;
}
```

**⚠️ Warning:** Không overuse

```css
/* ❌ Too many layers → memory overhead */
* {
  will-change: transform; /* Bad! */
}

/* ✅ Chỉ elements sắp animate */
.button:hover {
  will-change: transform;
}
```

---

### 🎯 CSS Containment – Isolate rendering scope

```css
.card {
  /* Browser biết card này independent */
  contain: layout style paint;
}

/* Khi card update → browser chỉ re-render card đó */
/* Không ảnh hưởng siblings/parents */
```

**Benefits:**
- Faster layout calculation
- Smaller paint areas
- Better performance với long lists

---

## 3.5 Intersection Observer – Lazy Load & Infinite Scroll

### 🎯 Intersection Observer Basics

**Replace scroll event listeners:**
```javascript
// ❌ Old way (performance issue)
window.addEventListener('scroll', () => {
  images.forEach(img => {
    const rect = img.getBoundingClientRect(); // Force layout!
    if (rect.top < window.innerHeight) {
      loadImage(img);
    }
  });
});

// ✅ New way (efficient)
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      loadImage(entry.target);
      observer.unobserve(entry.target);
    }
  });
});

images.forEach(img => observer.observe(img));
```

---

### 📋 Pattern 1: Lazy load images

```javascript
function setupLazyLoading() {
  const imageObserver = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const img = entry.target;
        
        // Load actual image
        img.src = img.dataset.src;
        img.classList.add('loaded');
        
        // Stop observing
        imageObserver.unobserve(img);
      }
    });
  }, {
    rootMargin: '50px' // Preload 50px before entering viewport
  });

  // Observe all lazy images
  document.querySelectorAll('img[data-src]').forEach(img => {
    imageObserver.observe(img);
  });
}

// HTML
// <img data-src="real-image.jpg" src="placeholder.jpg" alt="...">
```

---

### 📋 Pattern 2: Infinite scroll

```javascript
function InfiniteScroll({ items, loadMore, hasMore }) {
  const loadMoreRef = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasMore) {
          loadMore();
        }
      },
      { threshold: 0.5 } // Trigger khi 50% visible
    );

    if (loadMoreRef.current) {
      observer.observe(loadMoreRef.current);
    }

    return () => observer.disconnect();
  }, [loadMore, hasMore]);

  return (
    <div>
      {items.map(item => <Item key={item.id} {...item} />)}
      {hasMore && <div ref={loadMoreRef}>Loading...</div>}
    </div>
  );
}
```

---

### 📋 Pattern 3: Viewport tracking (analytics)

```javascript
// Track which elements user actually sees
const visibilityObserver = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        // Element became visible
        analytics.track('element_viewed', {
          element: entry.target.id,
          visibleDuration: Date.now()
        });
      } else {
        // Element left viewport
        analytics.track('element_left', {
          element: entry.target.id
        });
      }
    });
  },
  { threshold: 0.75 } // 75% visible
);

// Track important elements
document.querySelectorAll('[data-track-view]').forEach(el => {
  visibilityObserver.observe(el);
});
```

---

## 3.6 Virtual Scrolling – Render only visible items

### 🎯 Problem: Long Lists

```javascript
// ❌ Render 10000 items → DOM nodes = 10000
function ProductList({ products }) {
  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} {...product} />
      ))}
    </div>
  );
}

// 10000 DOM nodes = slow initial render + high memory
```

---

### ✅ Solution: React Virtual (react-window)

```javascript
import { FixedSizeList } from 'react-window';

function ProductList({ products }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      <ProductCard {...products[index]} />
    </div>
  );

  return (
    <FixedSizeList
      height={600}        // Container height
      itemCount={products.length}
      itemSize={120}      // Each item height
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// Only renders ~10 visible items at a time
// Scroll → reuse DOM nodes
// Memory: 10 nodes instead of 10000 ✅
```

---

### 📋 Variable Size List

```javascript
import { VariableSizeList } from 'react-window';

function ChatMessages({ messages }) {
  const listRef = useRef();
  const rowHeights = useRef({});

  function getRowHeight(index) {
    return rowHeights.current[index] || 80;
  }

  function setRowHeight(index, height) {
    if (rowHeights.current[index] !== height) {
      rowHeights.current[index] = height;
      listRef.current.resetAfterIndex(index);
    }
  }

  const Row = ({ index, style }) => {
    const rowRef = useRef();

    useEffect(() => {
      if (rowRef.current) {
        setRowHeight(index, rowRef.current.clientHeight);
      }
    }, [index]);

    return (
      <div style={style}>
        <div ref={rowRef}>
          <Message {...messages[index]} />
        </div>
      </div>
    );
  };

  return (
    <VariableSizeList
      ref={listRef}
      height={600}
      itemCount={messages.length}
      itemSize={getRowHeight}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  );
}
```

---

## 🎓 Tổng kết Runtime Performance

### ✅ Checklist – Optimize theo thứ tự

**Phase 1: JavaScript execution**
1. ☐ Debounce frequent events (search, auto-save)
2. ☐ Throttle continuous events (scroll, resize)
3. ☐ Use requestAnimationFrame cho animations
4. ☐ Offload heavy work to Web Workers

**Phase 2: Layout & Paint**
1. ☐ Tránh layout thrashing (batch reads/writes)
2. ☐ Use transform/opacity thay vì layout properties
3. ☐ Add will-change cho animating elements
4. ☐ Use CSS containment cho independent components

**Phase 3: Rendering optimization**
1. ☐ Intersection Observer thay scroll listeners
2. ☐ Lazy load images below fold
3. ☐ Virtual scrolling cho long lists (>100 items)
4. ☐ Defer below-the-fold content

---

### 📊 Impact Matrix – ROI của từng optimization

| Optimization | Effort | Impact | When to do |
|-------------|--------|--------|-----------|
| **Debounce/Throttle** | Low (30min) | High | Immediately |
| **requestAnimationFrame** | Low (1h) | High | For animations |
| **Avoid layout thrashing** | Medium (1 day) | High | If jank detected |
| **Use transform/opacity** | Low (2h) | High | For all animations |
| **Intersection Observer** | Low (2h) | High | For lazy loading |
| **Web Workers** | High (1 week) | Medium | For heavy computation |
| **Virtual scrolling** | Medium (2 days) | High | Lists >100 items |
| **CSS containment** | Low (1h) | Medium | Large component trees |

---

### 🎯 Decision Framework

```
Animation janky (<60fps)?
├─ Using layout properties? → Switch to transform/opacity
├─ No will-change? → Add will-change
└─ Heavy JS during animation? → Simplify or defer

Scroll laggy?
├─ Scroll listeners? → Use Intersection Observer
├─ Long list (>100)? → Virtual scrolling
└─ Images loading? → Lazy load

Input unresponsive?
├─ Heavy event handler? → Debounce/throttle
├─ Blocking computation? → Web Worker
└─ Re-rendering too much? → React.memo, useMemo
```

---

**DOCUMENT COMPLETE ✅**

*Version: 1.0*  
*Last updated: 2026-01-02*  
*Focus: RUNTIME PERFORMANCE*  
*Related: Measure Performance, Network Optimization, Memory Management*
