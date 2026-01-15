# PHẦN 2 — NETWORK OPTIMIZATION
Optimize những gì đi qua network, trước khi optimize runtime

## 2.1 Network Cost – Hiểu đúng trước khi optimize

### 📖 Bản chất của Network Performance

Senior FE thường nghĩ:
> "Network optimization = giảm file size"

**Không đúng.**

Network performance có **4 dimensions**:

#### 1️⃣ Latency (Độ trễ)
- **Round-trip time** (RTT) giữa client và server
- Không phụ thuộc vào bandwidth
- **Không thể compress**

```
Example:
- RTT: 100ms (Việt Nam → Singapore)
- 3 round trips = 300ms chỉ để handshake
- Chưa download gì cả

→ Optimize: Giảm requests, HTTP/2, HTTP/3
```

#### 2️⃣ Bandwidth (Băng thông)
- Volume data có thể transfer / second
- Throttle khác nhau: 4G vs WiFi vs 5G

```
4G (4 Mbps):
- 500KB file = 1000ms
- 100KB file = 200ms

Fast WiFi (100 Mbps):
- 500KB file = 40ms
- 100KB file = 8ms

→ Optimize: Compression, smaller assets
```

#### 3️⃣ Request Count
- Mỗi request có overhead (headers, handshake)
- HTTP/1.1: 6 concurrent requests max/domain
- HTTP/2: Multiplexing, nhưng vẫn có cost

```
100 requests × 50ms RTT = 5000ms (HTTP/1.1)
100 requests × ~200ms total (HTTP/2 multiplexing)

→ Optimize: Bundle, sprite, inline critical resources
```

#### 4️⃣ Cache Efficiency
- Không download lại = nhanh nhất
- Browser cache, CDN cache, Service Worker

```
First visit: 2MB download
Repeat visit with cache: 0KB download

→ Optimize: Cache headers, versioning, Service Worker
```

---

### 💡 Mental Model: Network Waterfall

**Không phải "download nhanh" là đủ.**

```
Critical rendering path:

HTML
 ↓ (blocking)
CSS ─────────────┐
                 ↓ (blocking)
              Render
                 ↓
              JavaScript (defer)
                 ↓
              Images (lazy)

Latency matters MORE than size cho critical path.
```

**Rule of thumb:**
- **Critical resources:** Minimize latency (preload, inline, HTTP/2 push)
- **Non-critical resources:** Defer, lazy load
- **Static resources:** Maximize cache

---

## 2.2 Bundle Optimization – Giảm JavaScript Cost

### 🎯 Tree Shaking – Loại bỏ code không dùng

**Concept:**
Build tool phân tích code → remove unused exports

**Điều kiện để tree-shaking work:**
1. ✅ ES modules (`import/export`)
2. ✅ `sideEffects` config trong package.json
3. ✅ Production build mode

#### Setup Tree Shaking

**Webpack:**
```javascript
// webpack.config.js
module.exports = {
  mode: 'production', // Enable tree shaking
  optimization: {
    usedExports: true,
    minimize: true
  }
};

// package.json
{
  "sideEffects": false // Enable aggressive tree-shaking
  // Hoặc specific files
  "sideEffects": ["*.css", "*.scss", "src/polyfills.js"]
}
```

**Vite:**
```javascript
// vite.config.js
export default {
  build: {
    minify: 'terser', // Tree shaking + minification
    terserOptions: {
      compress: {
        drop_console: true, // Remove console.log
        pure_funcs: ['console.info', 'console.debug']
      }
    }
  }
};
```

---

#### Pattern: Import chính xác, không import toàn bộ

```javascript
// ❌ Import toàn bộ (no tree-shaking)
import _ from 'lodash';
const debounced = _.debounce(fn, 100);
// → Bundle 70KB

import * as utils from './utils';
utils.formatDate(date);
// → Bundle toàn bộ utils

// ✅ Import specific (tree-shakeable)
import debounce from 'lodash-es/debounce';
// → Bundle 2KB

import { formatDate } from './utils';
// → Bundle chỉ formatDate
```

**Lý do:**
- Default import (`import _`) → bundle toàn bộ module
- Named import (`import { x }`) → tree-shakeable nếu module support

---

#### Pattern: Tránh barrel exports

```javascript
// ❌ components/index.js (barrel file)
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';
// ... 50 components

// ❌ Import từ barrel
import { Button } from '@/components';
// → Webpack vẫn parse toàn bộ index.js
// → Có thể bundle unused components

// ✅ Import direct
import { Button } from '@/components/Button';
// → Tree-shaking chắc chắn work
```

**Khi nào barrel file OK:**
- Library đã được built riêng từng module (e.g., `@mui/material`)
- Số lượng exports nhỏ (<10)

---

### 🎯 Code Splitting – Chia bundle thông minh

**Concept:**
Thay vì 1 bundle lớn, chia thành nhiều chunks nhỏ.  
Load chunk khi cần.

#### Strategy 1: Route-based Splitting

**React Router:**
```javascript
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// ❌ Static import
import Home from './pages/Home';
import Dashboard from './pages/Dashboard';
import Analytics from './pages/Analytics';

// ✅ Dynamic import (code splitting)
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Analytics = lazy(() => import('./pages/Analytics'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<Spinner />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/analytics" element={<Analytics />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**Impact:**
```
Before:
- main.js: 800KB (all routes)
- Initial load: 800KB

After:
- main.js: 150KB (core + home)
- dashboard.chunk.js: 300KB (load khi cần)
- analytics.chunk.js: 350KB (load khi cần)
- Initial load: 150KB ✅ (-81%)
```

---

#### Strategy 2: Component-based Splitting

```javascript
// ❌ Import heavy component statically
import HeavyChart from './HeavyChart'; // 250KB

function Dashboard() {
  return (
    <div>
      <Header />
      <HeavyChart data={data} />
    </div>
  );
}

// ✅ Dynamic import heavy component
import { lazy, Suspense } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <div>
      <Header />
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart data={data} />
      </Suspense>
    </div>
  );
}
```

**Rule of thumb:**
- Component > 50KB → consider splitting
- Component không phải "above the fold" → split
- Component dùng ít (modal, chart, editor) → split

---

#### Strategy 3: Vendor Splitting

```javascript
// webpack.config.js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      // React vendor chunk (stable, cacheable)
      react: {
        test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
        name: 'vendor-react',
        priority: 10
      },
      
      // UI library chunk
      ui: {
        test: /[\\/]node_modules[\\/](@mui)[\\/]/,
        name: 'vendor-ui',
        priority: 9
      },
      
      // Other vendors
      vendors: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendor-other',
        priority: 8
      }
    }
  }
}
```

**Benefit:**
```
main.js thay đổi thường xuyên → cache busted
vendor-react.js ít thay đổi → long-term cache ✅
```

---

#### Anti-pattern: Over-splitting

```javascript
// ❌ Split component quá nhỏ
const Button = lazy(() => import('./Button')); // 2KB
const Icon = lazy(() => import('./Icon'));     // 1KB

// Network overhead > savings
// Mỗi chunk có minimum overhead ~1-2KB
```

**Rule:** Minimum chunk size = 20KB

---

### 🎯 Lazy Loading – Load khi cần

#### Pattern 1: Lazy load images (Intersection Observer)

```javascript
// ✅ Native lazy loading (simplest)
<img 
  src="hero.jpg" 
  loading="lazy" 
  alt="Hero" 
/>

// ✅ Intersection Observer (more control)
import { useEffect, useRef, useState } from 'react';

function LazyImage({ src, alt }) {
  const [isLoaded, setIsLoaded] = useState(false);
  const imgRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsLoaded(true);
          observer.disconnect();
        }
      },
      { rootMargin: '50px' } // Preload 50px trước khi vào viewport
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, []);

  return (
    <img
      ref={imgRef}
      src={isLoaded ? src : 'placeholder.jpg'}
      alt={alt}
      className={isLoaded ? 'loaded' : 'loading'}
    />
  );
}
```

**Impact:**
```
Page có 50 images:
- Without lazy: Load 50 images (5MB) upfront
- With lazy: Load 3-5 visible images (~300KB) initially
```

---

#### Pattern 2: Lazy load below-the-fold content

```javascript
// Defer non-critical sections
function ProductPage() {
  const [showReviews, setShowReviews] = useState(false);

  useEffect(() => {
    // Load reviews after initial render
    const timer = setTimeout(() => setShowReviews(true), 1000);
    return () => clearTimeout(timer);
  }, []);

  return (
    <div>
      <ProductInfo /> {/* Critical: Load immediately */}
      <ProductImages /> {/* Critical */}
      
      {showReviews && (
        <Suspense fallback={<Skeleton />}>
          <ReviewSection /> {/* Non-critical: Deferred */}
        </Suspense>
      )}
    </div>
  );
}
```

---

#### Pattern 3: Conditional loading (feature flags)

```javascript
// ❌ Import modal upfront
import Modal from './Modal';

function App() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open</button>
      {isOpen && <Modal />}
    </>
  );
}

// ✅ Import modal only when needed
function App() {
  const [isOpen, setIsOpen] = useState(false);
  const [Modal, setModal] = useState(null);
  
  const handleOpen = async () => {
    if (!Modal) {
      const { default: ModalComponent } = await import('./Modal');
      setModal(() => ModalComponent);
    }
    setIsOpen(true);
  };
  
  return (
    <>
      <button onClick={handleOpen}>Open</button>
      {isOpen && Modal && <Modal />}
    </>
  );
}
```

---

## 2.3 Preloading & Prefetching – Optimize critical path

### 🎯 Resource Hints – Hint cho browser

#### 1. `dns-prefetch` – Resolve DNS sớm

```html
<!-- Resolve DNS cho third-party domains -->
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
<link rel="dns-prefetch" href="https://analytics.google.com">
```

**Khi nào dùng:**
- Third-party domains
- CDN domains
- API domains (nếu khác domain)

**Impact:**
- Save ~20-120ms DNS lookup
- Free (no bandwidth cost)

---

#### 2. `preconnect` – Handshake sớm

```html
<!-- Establish connection cho critical resources -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://cdn.example.com">
```

**Bao gồm:**
- DNS resolution
- TCP handshake
- TLS negotiation (HTTPS)

**Khi nào dùng:**
- Critical third-party resources (fonts, CDN)
- **Limit: 4-6 preconnect max** (browser limit)

**Impact:**
- Save ~100-300ms connection time
- Small CPU cost

---

#### 3. `preload` – Load critical resources sớm

```html
<!-- Preload critical CSS -->
<link rel="preload" href="/critical.css" as="style">

<!-- Preload critical fonts -->
<link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossorigin>

<!-- Preload hero image -->
<link rel="preload" href="/hero.jpg" as="image">

<!-- Preload critical JavaScript -->
<link rel="preload" href="/app.js" as="script">
```

**Khi nào dùng:**
- LCP element (hero image, critical text)
- Critical fonts (above-the-fold text)
- Critical CSS/JS không được discover sớm

**⚠️ Warning:** Preload tất cả = waste bandwidth

**Rule of thumb:**
- Preload max 2-3 resources
- Chỉ preload truly critical resources

---

#### 4. `prefetch` – Hint cho next navigation

```html
<!-- Prefetch next page likely visited -->
<link rel="prefetch" href="/dashboard.js">
<link rel="prefetch" href="/profile.js">
```

**Khi nào dùng:**
- Next page user có thể visit (e.g., login → dashboard)
- Low priority (browser load khi idle)

**React Example:**
```javascript
import { useEffect } from 'react';

function LoginPage() {
  useEffect(() => {
    // Prefetch dashboard chunks after login page loaded
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = '/dashboard.chunk.js';
    document.head.appendChild(link);
  }, []);

  return <LoginForm />;
}
```

---

### 📊 Decision Matrix: Which hint to use?

| Resource | When needed | Hint | Priority |
|----------|-------------|------|----------|
| **Third-party domain** | Setup connection | `dns-prefetch` or `preconnect` | Medium |
| **Critical font** | First paint | `preload` | High |
| **LCP image** | First paint | `preload` | High |
| **Critical CSS** | First paint | `preload` | High |
| **Next page asset** | Future navigation | `prefetch` | Low |
| **API data** | Future interaction | `prefetch` | Low |

---

### ⚠️ Common Mistakes

**1. Preload quá nhiều resources**
```html
<!-- ❌ Preload everything -->
<link rel="preload" href="/style1.css" as="style">
<link rel="preload" href="/style2.css" as="style">
<link rel="preload" href="/style3.css" as="style">
<link rel="preload" href="/image1.jpg" as="image">
<link rel="preload" href="/image2.jpg" as="image">

<!-- Bandwidth wasted, blocking other resources -->
```

**2. Preload non-critical resources**
```html
<!-- ❌ Preload below-the-fold image -->
<link rel="preload" href="/footer-logo.png" as="image">

<!-- ✅ Lazy load instead -->
<img src="/footer-logo.png" loading="lazy">
```

**3. Forget crossorigin cho fonts**
```html
<!-- ❌ Font preload không work -->
<link rel="preload" href="/font.woff2" as="font" type="font/woff2">

<!-- ✅ Must have crossorigin -->
<link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossorigin>
```

---

## 2.4 Service Workers – Ultimate caching layer

### 🎯 Service Worker Basics

**Concept:**
Proxy layer giữa app và network.  
Intercept requests → serve từ cache hoặc network.

**Use cases:**
- Offline support
- Faster repeat visits
- Background sync
- Push notifications

---

### 📋 Service Worker Caching Strategies

#### Strategy 1: Cache First (Fastest)

```javascript
// sw.js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cachedResponse) => {
      // Return cache if exists
      if (cachedResponse) {
        return cachedResponse;
      }
      
      // Otherwise fetch from network
      return fetch(event.request).then((networkResponse) => {
        // Cache for next time
        return caches.open('v1').then((cache) => {
          cache.put(event.request, networkResponse.clone());
          return networkResponse;
        });
      });
    })
  );
});
```

**Khi nào dùng:**
- Static assets (JS, CSS, fonts, images)
- Không thay đổi thường xuyên
- **Trade-off:** Có thể serve stale content

---

#### Strategy 2: Network First (Fresh data)

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    fetch(event.request)
      .then((networkResponse) => {
        // Update cache với response mới
        caches.open('v1').then((cache) => {
          cache.put(event.request, networkResponse.clone());
        });
        return networkResponse;
      })
      .catch(() => {
        // Fallback to cache if network fails
        return caches.match(event.request);
      })
  );
});
```

**Khi nào dùng:**
- API requests
- Dynamic content
- Offline fallback

---

#### Strategy 3: Stale-While-Revalidate (Best of both)

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open('v1').then((cache) => {
      return cache.match(event.request).then((cachedResponse) => {
        // Fetch từ network trong background
        const fetchPromise = fetch(event.request).then((networkResponse) => {
          cache.put(event.request, networkResponse.clone());
          return networkResponse;
        });

        // Return cache ngay lập tức nếu có
        // Hoặc đợi network
        return cachedResponse || fetchPromise;
      });
    })
  );
});
```

**Khi nào dùng:**
- Social feeds
- News content
- Balance speed + freshness

---

### 💡 Implementation với Workbox

**Thay vì viết SW thủ công, dùng Workbox (recommended):**

```javascript
// vite.config.js hoặc next.config.js
import { VitePWA } from 'vite-plugin-pwa';

export default {
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        // Cache static assets
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
            handler: 'CacheFirst',
            options: {
              cacheName: 'google-fonts-cache',
              expiration: {
                maxEntries: 10,
                maxAgeSeconds: 60 * 60 * 24 * 365 // 1 year
              }
            }
          },
          {
            urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp)$/i,
            handler: 'CacheFirst',
            options: {
              cacheName: 'images-cache',
              expiration: {
                maxEntries: 50,
                maxAgeSeconds: 60 * 60 * 24 * 30 // 30 days
              }
            }
          },
          {
            urlPattern: /\/api\/.*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              networkTimeoutSeconds: 3,
              expiration: {
                maxEntries: 50,
                maxAgeSeconds: 300 // 5 minutes
              }
            }
          }
        ]
      }
    })
  ]
};
```

---

### ⚠️ Service Worker Gotchas

**1. Cache invalidation**
```javascript
// Phải version cache
const CACHE_NAME = 'v1.2.0'; // Update khi deploy

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name)) // Delete old caches
      );
    })
  );
});
```

**2. Development vs Production**
```javascript
// ❌ SW active trong development = painful debugging
if (process.env.NODE_ENV === 'production') {
  navigator.serviceWorker.register('/sw.js');
}
```

**3. HTTPS only**
Service Worker chỉ work trên HTTPS (hoặc localhost).

---

## 2.5 CDN Usage – Distribute content globally

### 🎯 CDN Basics

**Concept:**
Serve static assets từ servers gần user nhất.

**Benefits:**
1. **Reduced latency:** 200ms RTT → 20ms RTT (SG user, SG CDN edge)
2. **Offload origin:** Static assets không hit main server
3. **DDoS protection:** CDN absorb traffic
4. **HTTP/2, HTTP/3:** Modern protocols

---

### 📋 CDN Best Practices

#### 1. Separate static assets domain

```
❌ https://example.com/bundle.js (same domain)
✅ https://cdn.example.com/bundle.js (CDN domain)
✅ https://static.example.com/bundle.js (CDN domain)
```

**Why:**
- Cookieless domain (giảm request size)
- Parallel downloads (browser limit per domain)
- Easier cache purging

---

#### 2. Long-term caching với versioning

```html
<!-- ❌ No versioning -->
<script src="https://cdn.example.com/app.js"></script>

<!-- Code update → phải cache bust toàn bộ CDN -->

<!-- ✅ Content hash versioning -->
<script src="https://cdn.example.com/app.a1b2c3d4.js"></script>

<!-- Code update → new filename → no cache issue -->
```

**Webpack auto-generate:**
```javascript
// webpack.config.js
output: {
  filename: '[name].[contenthash].js',
  chunkFilename: '[name].[contenthash].chunk.js'
}
```

**Cache headers:**
```
Cache-Control: public, max-age=31536000, immutable
```

- `max-age=31536000` = 1 year
- `immutable` = browser không revalidate (Chrome optimization)

---

#### 3. Separate cache policies

```
Static assets (versioned):
  Cache-Control: public, max-age=31536000, immutable
  
HTML (entry point):
  Cache-Control: no-cache
  // Always revalidate, but can use 304 Not Modified
  
API responses:
  Cache-Control: private, max-age=300
  // 5 minutes, không cache ở CDN
```

---

#### 4. Use CDN for third-party libraries

```html
<!-- ❌ Bundle React vào main.js -->
// main.js chứa React → bundle size +130KB

<!-- ✅ Load React từ CDN -->
<script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>

<!-- Hoặc -->
<script src="https://cdn.jsdelivr.net/npm/react@18"></script>
```

**Benefits:**
- Giảm bundle size
- Shared cache (nếu nhiều sites dùng cùng CDN URL)
- Faster delivery (CDN edge servers)

---

### 📊 CDN Providers Comparison

**Popular options:**

| Provider | Best for | Pros | Cons |
|----------|----------|------|------|
| **Cloudflare** | General, free tier | Free, easy setup, global | Limited config on free |
| **Fastly** | High-performance | Instant purge, real-time | Expensive |
| **AWS CloudFront** | AWS ecosystem | Tight integration, scalable | Complex pricing |
| **Vercel** | Next.js apps | Zero config, edge functions | Vendor lock-in |
| **Netlify** | Static sites | Simple, Git integration | Not specialized for high-traffic |

---

## 2.6 Compression – Giảm transfer size

### 🎯 Gzip vs Brotli

**Gzip:**
- Standard, supported everywhere
- Compression ratio: ~70% reduction
- Fast decompression

**Brotli:**
- Newer algorithm (2015)
- Compression ratio: ~20% better than gzip
- Supported: Chrome, Firefox, Edge, Safari

**Rule of thumb:**
- Serve Brotli cho modern browsers
- Fallback to Gzip cho old browsers

---

### 📋 Server Configuration

**Nginx:**
```nginx
# Gzip
gzip on;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;

# Brotli
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
```

**Node.js (Express):**
```javascript
const compression = require('compression');
const express = require('express');
const app = express();

app.use(compression({
  level: 6,
  threshold: 1024, // Only compress > 1KB
  filter: (req, res) => {
    // Compress all text-based responses
    return /json|text|javascript|css/.test(res.getHeader('Content-Type'));
  }
}));
```

---

### ⚠️ Compression Best Practices

**1. Pre-compress tại build time (tốt nhất)**
```javascript
// webpack.config.js
const CompressionPlugin = require('compression-webpack-plugin');

plugins: [
  // Generate .gz files
  new CompressionPlugin({
    filename: '[path][base].gz',
    algorithm: 'gzip',
    test: /\.(js|css|html|svg)$/,
    threshold: 1024,
    minRatio: 0.8
  }),
  
  // Generate .br files
  new CompressionPlugin({
    filename: '[path][base].br',
    algorithm: 'brotliCompress',
    test: /\.(js|css|html|svg)$/,
    threshold: 1024
  })
]
```

**Server serve pre-compressed:**
```nginx
location ~* \.(js|css)$ {
  gzip_static on;
  brotli_static on;
}
```

**Benefits:**
- Không compress on-the-fly (CPU saved)
- Better compression (có thể dùng max level tại build time)

---

**2. Không compress images/videos**
```javascript
// ❌ Compress JPEG/PNG
// Already compressed formats

// ✅ Compress SVG
// Text-based, benefit from compression
```

---

**3. Image format optimization**

Use modern formats:
```html
<picture>
  <source srcset="hero.avif" type="image/avif">
  <source srcset="hero.webp" type="image/webp">
  <img src="hero.jpg" alt="Hero">
</picture>
```

**Compression comparison:**
```
Same visual quality:
- JPEG: 100KB
- WebP: 50KB (-50%)
- AVIF: 30KB (-70%)
```

---

## 🎓 Tổng kết Network Optimization

### ✅ Checklist – Optimize theo thứ tự

**Phase 1: Measure (baseline)**
1. ☐ Lighthouse audit
2. ☐ Bundle analysis (`webpack-bundle-analyzer`)
3. ☐ Network waterfall (DevTools)
4. ☐ Field data (CWV, RUM)

**Phase 2: Quick wins**
1. ☐ Enable compression (Gzip/Brotli)
2. ☐ Optimize images (WebP, lazy loading)
3. ☐ Add cache headers
4. ☐ Use CDN

**Phase 3: Bundle optimization**
1. ☐ Tree shaking enabled
2. ☐ Code splitting (routes, components)
3. ☐ Remove duplicate dependencies
4. ☐ Replace heavy libraries

**Phase 4: Critical path**
1. ☐ Preload LCP element
2. ☐ Preconnect third-parties
3. ☐ Inline critical CSS
4. ☐ Defer non-critical JS

**Phase 5: Advanced**
1. ☐ Service Worker (offline, cache)
2. ☐ HTTP/2 or HTTP/3
3. ☐ Resource hints (prefetch next pages)
4. ☐ Edge computing (Cloudflare Workers)

---

### 📊 Impact Matrix – ROI của từng optimization

| Optimization | Effort | Impact | When to do |
|-------------|--------|--------|-----------|
| **Compression** | Low (1h) | High | Immediately |
| **Image optimization** | Low (2h) | High | Immediately |
| **CDN** | Low (1 day) | High | Early |
| **Code splitting** | Medium (1 week) | High | After bundle analysis |
| **Tree shaking** | Low (1 day) | Medium | Early |
| **Preload critical** | Low (2h) | Medium | After LCP identified |
| **Service Worker** | High (2 weeks) | Medium | After basics |
| **HTTP/2** | Low (IT setup) | Medium | Infrastructure upgrade |

---

### 🎯 Decision Framework

```
Bundle > 500KB?
├─ YES → Code splitting + tree shaking (priority 1)
└─ NO → Continue

LCP > 2.5s + Network bound?
├─ YES → Preload LCP element + CDN (priority 1)
└─ NO → Continue

Images > 1MB total?
├─ YES → Image optimization + lazy loading (priority 1)
└─ NO → Continue

Repeat visitors > 50%?
├─ YES → Service Worker caching (priority 2)
└─ NO → Focus on first-time performance
```

---

### 💡 Advanced Topics (Future Learning)

- **HTTP/3 (QUIC):** Faster than HTTP/2 (multiplexing without head-of-line blocking)
- **Edge computing:** Run code tại CDN edge (Cloudflare Workers, Vercel Edge Functions)
- **Module Federation:** Share code across micro-frontends
- **Streaming SSR:** Progressive HTML rendering

---

**DOCUMENT COMPLETE ✅**

*Version: 1.0*  
*Last updated: 2026-01-02*  
*Focus: NETWORK OPTIMIZATION (not measurement)*  
*Related: Measure Performance, Runtime Performance, Memory Management*
