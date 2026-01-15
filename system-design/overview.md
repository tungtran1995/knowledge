# System Design for Frontend – Overview

> **Scope:** Architecting large-scale frontend applications for scalability, performance, and maintainability.

---

## 1. What is Frontend System Design?

### Traditional vs Frontend System Design

**Backend System Design:**
- Databases, caching, load balancers
- Microservices, message queues
- CAP theorem, consistency models

**Frontend System Design:**
- Component architecture, state management
- Code splitting, lazy loading, caching
- Performance optimization at scale
- User experience at millions of users

---

## 2. Core Principles

### Principle 1: Progressive Enhancement

```
Layer 1: Core functionality (HTML, essential JS)
Layer 2: Enhanced UX (React, interactions)
Layer 3: Advanced features (offline, real-time)

Every layer works without next layer
```

**Why:** Works on slow networks, old devices, JS disabled.

---

### Principle 2: Performance Budget

```
Initial load: < 3s on 3G
Time to Interactive: < 5s
Bundle size: < 200KB (gzipped)
Lighthouse score: > 90
```

**Enforce:** Fail CI if budget exceeded.

---

### Principle 3: Fault Tolerance

```
API down? → Show cached data + offline banner
CDN slow? → Fallback to origin
JS error? → Error boundary, page still usable
```

**Never:** White screen of death.

---

### Principle 4: Scalability Mindset

```
1 user → 1,000 users → 1,000,000 users

What breaks at each scale?
- State management strategy
- Data fetching patterns
- Bundle size
- Real-time connections
```

---

## 3. System Design Topics

### Data Flow Architecture

**Problem:** How does data move through your app?

**Patterns:**
- Unidirectional (Redux, Zustand)
- Bidirectional (MobX, Vue)
- Event-driven (EventEmitter, PubSub)
- Reactive (RxJS, signals)

**Decision factors:**
- Team size, complexity, performance needs

---

### Caching Strategies

**Layers:**
```
Browser cache (static assets)
  ↓
Service Worker cache (offline)
  ↓
In-memory cache (React Query)
  ↓
API
```

**Strategies:**
- Cache-first (offline-first apps)
- Network-first (real-time data)
- Stale-while-revalidate (best UX)

---

### Code Organization

**Horizontal (by type):**
```
components/
hooks/
utils/
```

**Vertical (by feature):**
```
features/
  auth/
    components/
    hooks/
    api/
  dashboard/
    components/
    hooks/
    api/
```

**Recommendation:** Vertical for apps >10K LOC.

---

### State Management Architecture

**Layers:**
```
Server State (React Query)
  ↓
Global Client State (Zustand/Redux)
  ↓
Local Component State (useState)
  ↓
URL State (React Router)
```

**Rule:** Right tool for right layer.

---

## 4. Scalability Patterns

### Pattern 1: Code Splitting by Route

```javascript
// Load routes lazily
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

// User only downloads what they need
```

**Impact:** Initial bundle 500KB → 150KB.

---

### Pattern 2: Virtual Scrolling

```
Problem: Render 10,000 items → Slow
Solution: Only render visible items (50)

Libraries: react-window, react-virtualized
```

---

### Pattern 3: Incremental Static Generation (ISR)

```
Generate popular pages at build time
Generate rare pages on-demand
Revalidate stale pages in background

Next.js built-in
```

---

### Pattern 4: Edge Computing

```
Deploy compute close to users
CDN edge functions (Cloudflare Workers)
Reduce latency globally
```

---

## 5. Real-World Constraints

### Constraint 1: Network

**3G = 50% of global traffic**

**Solutions:**
- Aggressive code splitting
- Image optimization (WebP, lazy load)
- Resource hints (preload, prefetch)

---

### Constraint 2: Devices

**Low-end devices = 30% of users**

**Solutions:**
- Reduce JS execution time
- Avoid heavy animations
- Test on real devices (not just dev MacBook)

---

### Constraint 3: Browsers

**Old browsers still exist**

**Solutions:**
- Polyfills for critical features
- Progressive enhancement
- Graceful degradation

---

## 6. Key Metrics

### Performance Metrics

```
FCP (First Contentful Paint): < 1.8s
LCP (Largest Contentful Paint): < 2.5s
FID (First Input Delay): < 100ms
CLS (Cumulative Layout Shift): < 0.1
TTI (Time to Interactive): < 3.8s
```

---

### Business Metrics

```
Conversion rate impact:
- 1s delay = 7% drop in conversions
- 100ms improvement = 1% revenue increase

User retention:
- Slow app = high bounce rate
- Fast app = happy users
```

---

## 7. Interview Questions

### Common Questions:

**Q1:** Design a news feed like Twitter
**Q2:** Design a real-time chat application
**Q3:** Design a dashboard with 100K data points
**Q4:** Design an e-commerce site for global users
**Q5:** Design a collaborative editor (Google Docs)

**Approach:**
1. Requirements (functional, non-functional)
2. Architecture (high-level components)
3. Data model (entities, relationships)
4. API design (endpoints, real-time)
5. Performance (caching, optimization)
6. Trade-offs (discuss alternatives)

---

## 8. Tools & Frameworks

### Monitoring

```
Performance: Lighthouse, WebPageTest
Errors: Sentry, LogRocket
Analytics: Google Analytics, Mixpanel
Real User Monitoring: SpeedCurve, Calibre
```

---

### Architecture

```
Monolith: Next.js, Remix
Micro-frontends: Module Federation, Single-SPA
Component libraries: Storybook
Design systems: Figma + code gen
```

---

## Tổng Kết

**Frontend System Design = Architect for:**
- Performance (fast load, smooth interactions)
- Scalability (1 user → 1M users)
- Reliability (fault tolerance, offline)
- Maintainability (clean architecture)
- User Experience (always priority #1)

**Key Skills:**
1. Trade-off analysis
2. Performance optimization
3. Architecture patterns
4. Real-world constraints

---

**Next Topics:**
- [Data Flow →](./data-flow.md)
- [Caching Strategies →](./caching.md)
- [Scalability Patterns →](./scalability.md)
- [Real-World Examples →](./examples.md)
