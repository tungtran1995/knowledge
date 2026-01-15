# Micro-frontends 

---

## Concept - Phân tách Frontend như Microservices

### Nguồn gốc

**Inspiration**: Microservices architecture (Backend, 2014+)

**Core idea**: Thay vì 1 monolithic frontend app, chia thành nhiều **independent apps** được teams khác nhau develop, deploy độc lập, nhưng compose lại thành 1 UI thống nhất cho user.

```
Traditional (Monolith):
┌─────────────────────────────┐
│   Single Frontend App       │
│  (Header, Products, Cart,   │
│   Checkout, Profile...)     │
└─────────────────────────────┘
     ↓
  1 team maintain
  1 repo
  1 deployment

Micro-frontends:
┌─────────┐ ┌──────────┐ ┌─────────┐ ┌──────────┐
│ Header  │ │ Products │ │  Cart   │ │ Checkout │
│  Team A │ │  Team B  │ │ Team C  │ │  Team D  │
└─────────┘ └──────────┘ └─────────┘ └──────────┘
     ↓           ↓            ↓            ↓
  Deploy      Deploy       Deploy       Deploy
 independently independently independently independently
     ↓           ↓            ↓            ↓
        Compose thành 1 UI cho user
```

---

### Bản chất: Organizational scaling pattern

**Mental model**: Shopping mall
```
Monolith = 1 mega store (everything inside)
  → Hard to manage
  → 1 renovation = close whole store
  → All staff must coordinate

Micro-frontends = Multiple small shops
  → Each shop độc lập
  → Renovate 1 shop = others still open
  → Each shop has own staff
  → Mall provides: Location, parking, security (shared infrastructure)
```

**Triết lý**:
```
Conway's Law: 
"Architecture mirrors organization structure"

Large company (1000+ engineers):
→ Multiple teams, different priorities
→ Monolith = bottlenecks, conflicts, slow
→ Micro-frontends = teams autonomous
```

---

### Characteristics của Micro-frontend

#### 1. **Independent Deployment**
```
Team A deploy Header → No impact Team B's Products
Team B deploy Products → No impact Team C's Cart

Không cần coordinate releases
No "big bang" deployment
```

#### 2. **Technology Agnostic**
```
Header: React
Products: Vue
Cart: Angular
Checkout: Svelte

Each team chọn tech stack phù hợp
(Though trong thực tế, shared tech = better DX)
```

#### 3. **Team Ownership**
```
Team A owns:
- Header code
- Header deployment
- Header monitoring
- Header bugs

End-to-end ownership
```

#### 4. **Loosely Coupled**
```
Micro-frontends communicate qua:
- URL routing
- Events (CustomEvents)
- Shared state (minimal)

Not direct function calls
```

#### 5. **Isolated Development**
```
Each micro-frontend có:
- Own repo (hoặc monorepo với clear boundaries)
- Own dev server
- Own tests
- Own CI/CD

Can develop without running others
```

---

## Khi nào dùng Micro-frontends?

### ✅ Use when:

#### 1. **Large teams (50+ frontend engineers)**
```
Problem: 50 devs trên 1 codebase
- Merge conflicts daily
- Code review bottleneck
- Deploy queue (1 team block others)
- Shared dependencies hell

Solution: Micro-frontends
- 5 teams × 10 devs
- Each team own micro-frontend
- Independent deploys
- Less conflicts
```

#### 2. **Multiple business domains**
```
E-commerce platform:
- Product catalog (Team A)
- Shopping cart (Team B)
- Checkout (Team C)
- User profile (Team D)

Each domain có:
- Different update frequency
- Different complexity
- Different priorities

→ Micro-frontends align với domains
```

#### 3. **Different release cycles**
```
Problem: Marketing muốn deploy landing page daily
         Core app deploy weekly (strict QA)

Monolith: Marketing blocked

Micro-frontends: 
- Landing pages = separate micro-frontend
- Deploy freely
- Core app unaffected
```

#### 4. **Gradual migration**
```
Legacy app (Angular.js) → Modern (React)

Monolith: Rewrite toàn bộ (risk cao, time lâu)

Micro-frontends:
- Migrate từng page
- New pages = React micro-frontend
- Old pages = Angular micro-frontend
- Co-exist trong transition period
```

#### 5. **Independent scaling**
```
Black Friday: Product pages traffic 100x
              Other pages normal

With micro-frontends:
- Scale Product micro-frontend only
- Cost efficient
```

---

### ❌ Don't use when:

#### 1. **Small team (<10 devs)**
```
Overhead > Benefits
- 1 team có thể coordinate easily
- Deployment pipeline complexity not worth
- Shared component reuse harder
```

#### 2. **Simple apps**
```
Marketing site, Blog, Internal tool
- Monolith đủ
- Micro-frontends add unnecessary complexity
```

#### 3. **High cohesion requirements**
```
App với features deeply integrated:
- Real-time collaboration (Google Docs)
- Heavy shared state
- Complex interactions giữa components

→ Splitting = artificial boundaries
```

#### 4. **Performance critical**
```
Micro-frontends có overhead:
- Multiple bundle downloads
- Duplicate dependencies
- Runtime composition cost

Gaming, high-performance visualization → Monolith better
```

---

## Implementation Approaches

---

## 1. Module Federation (Webpack 5)

### Concept

**Core idea**: Share JavaScript modules **at runtime** giữa các apps, không cần rebuild.

```
Traditional:
App A build → bundle A (includes React)
App B build → bundle B (includes React)
User load → download React 2 times

Module Federation:
App A build → bundle A (exposes components)
App B build → bundle B (consumes A's components)
User load → download React 1 time (shared)
           → A và B share React instance
```

**Mental model**: Shared library nhưng at runtime, không phải build-time.

---

### Architecture

```
┌─────────────────┐
│   Host App      │ (Main app, load others)
│   (Shell)       │
└────────┬────────┘
         │ Load at runtime
    ┌────┴─────┬──────────┐
    ↓          ↓          ↓
┌────────┐ ┌────────┐ ┌────────┐
│ Remote │ │ Remote │ │ Remote │
│  App A │ │  App B │ │  App C │
└────────┘ └────────┘ └────────┘
(Products)  (Cart)    (Checkout)
```

**Roles**:
- **Host**: App chính, consume remotes
- **Remote**: Micro-frontend, expose modules

---

### Configuration

#### Remote App (Products)
```javascript
// webpack.config.js - Products micro-frontend
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'products',
      filename: 'remoteEntry.js',
      
      // Expose components
      exposes: {
        './ProductList': './src/ProductList',
        './ProductDetail': './src/ProductDetail',
      },
      
      // Share dependencies
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};
```

**Key concepts**:
- `name`: Remote name
- `exposes`: Modules khác apps có thể import
- `shared`: Dependencies share với host (tránh duplicate)

---

#### Host App (Shell)
```javascript
// webpack.config.js - Host app
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      
      // Declare remotes
      remotes: {
        products: 'products@http://localhost:3001/remoteEntry.js',
        cart: 'cart@http://localhost:3002/remoteEntry.js',
      },
      
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};
```

---

#### Usage trong Host
```javascript
// Host app dynamically import remote
import React, { lazy, Suspense } from 'react';

// Lazy load remote component
const ProductList = lazy(() => import('products/ProductList'));
const Cart = lazy(() => import('cart/CartComponent'));

function App() {
  return (
    <div>
      <Header />
      
      <Suspense fallback={<div>Loading...</div>}>
        <ProductList />
      </Suspense>
      
      <Suspense fallback={<div>Loading cart...</div>}>
        <Cart />
      </Suspense>
    </div>
  );
}
```

---

### Shared Dependencies - Chi tiết

**Problem**: Duplicate dependencies
```
Without sharing:
Products bundle: React (40KB) + ProductList code
Cart bundle: React (40KB) + Cart code
→ User download React 2 times
```

**Solution**: Singleton shared
```javascript
shared: {
  react: { 
    singleton: true,     // Only 1 instance
    requiredVersion: '^18.0.0',
  },
}
```

**Runtime behavior**:
```
1. Host loads, download React 18.2.0
2. Products loads, check: Host has React? 
   → Yes → Reuse host's React
   → No → Download own React
```

**Version conflicts**:
```javascript
Host: React 18.2.0
Remote: React 17.0.0

Option 1: singleton: true
→ Remote use Host's React 18 (may break if incompatible)

Option 2: singleton: false
→ Both versions loaded (larger bundle)

Real world: Align versions across teams (best)
```

---

### Benefits của Module Federation

#### 1. **True independent deployment**
```
Deploy Products micro-frontend:
1. Build Products
2. Upload to CDN (http://cdn.com/products/v123/remoteEntry.js)
3. Update remotes URL trong Host config
4. Done - Host auto load new version

No rebuild Host
```

#### 2. **Code sharing**
```
Shared:
- React, React-DOM (singleton)
- Design system components
- Utilities

Avoid:
- Duplicate downloads
- Version conflicts
```

#### 3. **Dynamic loading**
```
Load remotes on-demand:
- User visit /products → Load products remote
- User visit /cart → Load cart remote

Not visit /checkout → Never load checkout remote

→ Faster initial load
```

---

### Drawbacks

#### 1. **Complex debugging**
```
Error trong remote component:
- Source maps across apps
- Stack traces reference remote URLs
- Hard to trace bugs

Mitigation: Good logging, monitoring
```

#### 2. **Runtime overhead**
```
Host phải:
- Fetch remoteEntry.js
- Parse module definitions
- Load chunks on-demand

→ Slower than monolith (nhưng offset bởi code splitting)
```

#### 3. **Network dependency**
```
Remote URL down → Feature broken

Mitigation:
- Fallback UI
- Health checks
- CDN với high availability
```

#### 4. **Version management**
```
Host React 18
Remote A React 18.2
Remote B React 18.1

Shared singleton → Conflicts possible

Solution: Team coordination, shared deps policy
```

---

### Best Practices

#### 1. **Version alignment**
```
All teams dùng same major version:
- React 18.x (not mix 17 và 18)
- TypeScript 5.x

Use workspaces/monorepo để enforce
```

#### 2. **Clear contracts**
```
Remote expose clear API:
- TypeScript definitions
- Props interface
- Documentation

Example:
// ProductList.d.ts
export interface ProductListProps {
  category?: string;
  onProductClick: (id: string) => void;
}
```

#### 3. **Fallback UI**
```javascript
<Suspense fallback={<Skeleton />}>
  <ErrorBoundary fallback={<ErrorState />}>
    <RemoteComponent />
  </ErrorBoundary>
</Suspense>
```

#### 4. **Monitoring**
```javascript
// Track remote load failures
window.addEventListener('error', (event) => {
  if (event.filename?.includes('remoteEntry.js')) {
    analytics.track('remote_load_failed', {
      remote: event.filename,
    });
  }
});
```

---

## 2. iframe Approach

### Concept

**Core idea**: Mỗi micro-frontend chạy trong `<iframe>`, hoàn toàn isolated.

```html
<body>
  <iframe src="http://header.example.com"></iframe>
  <iframe src="http://products.example.com"></iframe>
  <iframe src="http://cart.example.com"></iframe>
</body>
```

**Mental model**: Windows trong OS
```
Mỗi iframe = 1 window riêng
- Own JavaScript context
- Own CSS scope
- Own document
- Cannot interfere với nhau
```

---

### Characteristics

#### 1. **Complete isolation**
```
Iframe A:
- Global variables: window.myVar
- CSS: .button { color: red }
- JavaScript errors

Iframe B:
- Không bị affect
- Own window.myVar
- Own .button styles
- Errors không crash parent
```

#### 2. **Security sandbox**
```html
<iframe 
  src="http://untrusted.com"
  sandbox="allow-scripts allow-same-origin"
></iframe>
```

**Sandbox options**:
- `allow-scripts`: Cho phép JavaScript
- `allow-same-origin`: Cho phép cookies
- `allow-forms`: Cho phép submit forms
- `allow-popups`: Cho phép popups

---

### Communication giữa iframes

#### Method 1: postMessage (Standard)
```javascript
// Parent → iframe
const iframe = document.getElementById('products-iframe');
iframe.contentWindow.postMessage(
  { type: 'UPDATE_FILTER', filter: 'electronics' },
  'http://products.example.com'
);

// Iframe nhận message
window.addEventListener('message', (event) => {
  if (event.origin !== 'http://parent.example.com') return;
  
  if (event.data.type === 'UPDATE_FILTER') {
    applyFilter(event.data.filter);
  }
});
```

#### Method 2: URL parameters
```javascript
// Parent update iframe URL
iframe.src = 'http://products.example.com?category=electronics';

// Iframe read params
const params = new URLSearchParams(window.location.search);
const category = params.get('category');
```

#### Method 3: Shared localStorage (Same origin only)
```javascript
// Parent write
localStorage.setItem('user', JSON.stringify(user));

// Iframe read (nếu same origin)
const user = JSON.parse(localStorage.getItem('user'));

// Listen changes
window.addEventListener('storage', (event) => {
  if (event.key === 'user') {
    updateUser(JSON.parse(event.newValue));
  }
});
```

---

### Benefits của iframe

#### 1. **True isolation**
```
Pros:
- CSS không conflict (mỗi iframe own styles)
- JavaScript errors không crash toàn page
- Global variables không collide
- Security (sandbox)

Use case: Embed 3rd-party widgets safely
```

#### 2. **Technology agnostic**
```
Parent: React
Iframe A: Vue
Iframe B: Angular
Iframe C: Plain HTML

→ No conflicts, each iframe độc lập
```

#### 3. **Independent deployment**
```
Update iframe content:
- Deploy new version
- Change iframe URL
- Done

Parent không cần rebuild
```

#### 4. **Security**
```
Embed untrusted content:
<iframe sandbox src="http://ads.thirdparty.com"></iframe>

→ Cannot access parent DOM
→ Cannot steal cookies
→ Cannot run malicious scripts (if no allow-scripts)
```

---

### Drawbacks của iframe

#### 1. **Poor UX**
```
Problems:
- Each iframe = separate page
- Separate scrollbars (nếu không fix height)
- Separate back/forward history
- Jumpy layout (iframe load async)
- Separate focus management
```

**Example nightmare**:
```
User scroll page → Chạm iframe → Mouse wheel scroll iframe, không scroll page
User tab → Focus nhảy vào iframe → Tab navigation bị break
```

#### 2. **Performance overhead**
```
Each iframe:
- Own document
- Own JavaScript context
- Own CSS engine
- Own layout engine

3 iframes = 3× overhead

Initial load slow (3 separate pages)
Memory usage high
```

#### 3. **Responsive design hell**
```
Parent responsive, iframe fixed size → Break on mobile

Solutions:
1. iframe height = content height (require postMessage)
2. iframe 100% width (still need handle height)
3. ResizeObserver (parent listen iframe size changes)

→ Complex
```

#### 4. **SEO problems**
```
Search engines:
- Không crawl iframe content tốt
- Iframe content = separate URL
- Parent page SEO không benefit từ iframe content

Marketing sites: Don't use iframes
```

#### 5. **Communication overhead**
```
postMessage = async, serialized

Parent → Iframe: postMessage({ type: 'UPDATE', data })
→ Serialize object
→ Send via message queue
→ Iframe receive
→ Deserialize

Slow so với direct function call

Cannot pass:
- Functions
- DOM nodes  
- Complex objects (circular refs)
```

---

### Khi nào dùng iframe?

#### ✅ Use when:

**1. Embed 3rd-party content**
```
Ads, widgets, payment forms
→ Security isolation critical
→ Don't trust 3rd-party code

Example: Stripe payment form
<iframe src="https://checkout.stripe.com/..."></iframe>
```

**2. Legacy migration**
```
Migrate từng page từ legacy (ASP.NET) → Modern (React)

Phase 1: Wrap legacy pages trong iframe
Phase 2: Rewrite 1 page → replace iframe với React
Phase 3: Repeat

Gradual, low-risk
```

**3. Multi-tenant apps**
```
White-label platform:
- Each tenant có custom UI
- Hosted on separate domains
- Parent shell load tenant iframes

Security: Tenants cannot access each other
```

#### ❌ Don't use when:

```
1. Performance critical (SPAs, dashboards)
2. Need tight integration (shared state, frequent communication)
3. SEO important (marketing sites)
4. Mobile apps (poor UX)
5. Modern stack (Module Federation better)
```

---

## 3. Web Components

### Concept

**Core idea**: Browser-native custom elements, work với mọi framework (React, Vue, Angular, vanilla).

```html
<!-- Use custom element like HTML tag -->
<product-card 
  product-id="123"
  price="29.99"
></product-card>

<!-- Web Component encapsulated -->
```

**Mental model**: Like `<video>`, `<audio>` tags
```
<video> = browser-native component
- Encapsulated (own controls, state)
- Works everywhere
- Standard API

<product-card> = custom component
- Same principles
- Framework agnostic
```

---

### Web Components APIs

#### 1. **Custom Elements** - Define custom tags
```javascript
class ProductCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }
  
  connectedCallback() {
    // Element added to DOM
    this.render();
  }
  
  disconnectedCallback() {
    // Element removed from DOM
    this.cleanup();
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    // Attribute changed
    if (name === 'price') {
      this.updatePrice(newValue);
    }
  }
  
  static get observedAttributes() {
    return ['price', 'product-id'];
  }
  
  render() {
    this.shadowRoot.innerHTML = `
      <style>
        .card { border: 1px solid #ccc; }
      </style>
      <div class="card">
        <h3>${this.getAttribute('product-id')}</h3>
        <p>$${this.getAttribute('price')}</p>
      </div>
    `;
  }
}

// Register custom element
customElements.define('product-card', ProductCard);
```

---

#### 2. **Shadow DOM** - CSS/DOM encapsulation
```javascript
class MyComponent extends HTMLElement {
  constructor() {
    super();
    // Create shadow root
    this.attachShadow({ mode: 'open' });
    
    this.shadowRoot.innerHTML = `
      <style>
        /* Styles ONLY affect shadow DOM */
        p { color: red; }
      </style>
      <p>This is red</p>
    `;
  }
}
```

**Isolation**:
```html
<style>
  /* Global style */
  p { color: blue; }
</style>

<p>This is blue (global)</p>

<my-component>
  <!-- Shadow DOM -->
  <p>This is red (isolated)</p>
</my-component>
```

---

#### 3. **HTML Templates** - Reusable markup
```html
<template id="product-card-template">
  <style>
    .card { border: 1px solid #ddd; }
  </style>
  <div class="card">
    <slot name="title"></slot>
    <slot name="price"></slot>
  </div>
</template>

<script>
class ProductCard extends HTMLElement {
  constructor() {
    super();
    const template = document.getElementById('product-card-template');
    const content = template.content.cloneNode(true);
    this.attachShadow({ mode: 'open' }).appendChild(content);
  }
}
customElements.define('product-card', ProductCard);
</script>

<!-- Usage -->
<product-card>
  <h3 slot="title">Product Name</h3>
  <p slot="price">$29.99</p>
</product-card>
```

---

### Usage trong Micro-frontends

#### Scenario: Multi-framework app
```
Shell: Plain HTML/JS
Micro-frontend A: React (build as Web Component)
Micro-frontend B: Vue (build as Web Component)
Micro-frontend C: Svelte (build as Web Component)
```

#### Build React as Web Component
```javascript
// products-micro-frontend (React)
import React from 'react';
import ReactDOM from 'react-dom/client';
import ProductList from './ProductList';

class ProductsElement extends HTMLElement {
  connectedCallback() {
    const root = ReactDOM.createRoot(this);
    root.render(<ProductList />);
  }
}

customElements.define('products-list', ProductsElement);
```

#### Usage trong Shell
```html
<!-- Shell app (framework agnostic) -->
<html>
<body>
  <!-- Load micro-frontends -->
  <script src="http://products.example.com/products.js"></script>
  <script src="http://cart.example.com/cart.js"></script>
  
  <!-- Use as HTML tags -->
  <products-list category="electronics"></products-list>
  <shopping-cart></shopping-cart>
</body>
</html>
```

---

### Communication patterns

#### 1. **Attributes (Parent → Child)**
```html
<product-list category="electronics" sort="price"></product-list>
```

```javascript
class ProductList extends HTMLElement {
  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'category') {
      this.loadProducts(newValue);
    }
  }
  
  static get observedAttributes() {
    return ['category', 'sort'];
  }
}
```

---

#### 2. **Custom Events (Child → Parent)**
```javascript
// Child dispatch event
class ProductCard extends HTMLElement {
  handleClick() {
    this.dispatchEvent(new CustomEvent('product-selected', {
      detail: { productId: this.productId },
      bubbles: true,
      composed: true, // Cross shadow DOM boundary
    }));
  }
}

// Parent listen
document.addEventListener('product-selected', (event) => {
  console.log('Selected:', event.detail.productId);
});
```

---

#### 3. **Shared Event Bus**
```javascript
// Shared bus
class EventBus extends EventTarget {}
window.appEventBus = new EventBus();

// Micro-frontend A dispatch
window.appEventBus.dispatchEvent(new CustomEvent('cart-updated', {
  detail: { itemCount: 5 }
}));

// Micro-frontend B listen
window.appEventBus.addEventListener('cart-updated', (event) => {
  updateCartBadge(event.detail.itemCount);
});
```

---

### Benefits của Web Components

#### 1. **Framework agnostic**
```
Build once → Use everywhere:
- React app
- Vue app
- Angular app
- Plain HTML

No vendor lock-in
```

#### 2. **Encapsulation**
```
Shadow DOM:
- CSS không leak out
- Global CSS không affect component
- True isolation

Good for: Design system components
```

#### 3. **Browser native**
```
No framework overhead
No build step required (optional)
Fast (browser-optimized)
```

#### 4. **Future-proof**
```
Web standards
Browsers evolve, components still work
No framework migrations
```

---

### Drawbacks của Web Components

#### 1. **Poor DX (Developer Experience)**
```
Vanilla Web Components:
- No JSX (template strings)
- No reactive state (manual updates)
- No dev tools
- Verbose lifecycle

vs React:
- JSX, hooks, DevTools
- Much better DX
```

#### 2. **Framework features missing**
```
No built-in:
- State management
- Routing
- Server-side rendering (SSR)
- Code splitting

Must implement manually or use libraries
```

#### 3. **Shadow DOM limitations**
```
Problems:
- Global styles không apply (must redefine inside)
- Form elements tricky (form.elements không see shadow)
- Accessibility issues (ARIA, focus management)
- DevTools inspect harder
```

#### 4. **Polyfills for old browsers**
```
IE11, old Safari không support
→ Need polyfills (~30KB)
→ Performance hit
```

#### 5. **Hydration issues (SSR)**
```
Server render Web Components hard
- Declarative Shadow DOM (new, limited support)
- Or client-side only (bad for SEO)
```

---

### Khi nào dùng Web Components?

#### ✅ Use when:

**1. Design system cross-framework**
```
Company có:
- Legacy app (Angular)
- New app (React)
- Marketing site (Vue)

Build design system as Web Components
→ All apps dùng same components
```

**2. Micro-frontends với different frameworks**
```
Teams tự do chọn framework
Build features as Web Components
Shell compose lại
```

**3. Widgets for external use**
```
Build embeddable widget (chat, analytics)
Customers có thể embed vào any site
<script src="widget.js"></script>
<my-widget api-key="..."></my-widget>
```

#### ❌ Don't use when:

```
1. Single framework app (React hooks > Web Components)
2. Need SSR (Web Components SSR complex)
3. Complex state management (better solutions exist)
4. Team không familiar (learning curve)
```

---

### Modern approach: Lit (Web Components library)
```javascript
import { LitElement, html, css } from 'lit';

class ProductCard extends LitElement {
  static properties = {
    productId: { type: String },
    price: { type: Number },
  };
  
  static styles = css`
    .card { border: 1px solid #ddd; }
  `;
  
  render() {
    return html`
      <div class="card">
        <h3>${this.productId}</h3>
        <p>$${this.price}</p>
      </div>
    `;
  }
}

customElements.define('product-card', ProductCard);
```

**Benefits of Lit**:
- Reactive properties (like React state)
- Better DX (html tagged template)
- Small (~5KB)
- Still standard Web Components

---

## Trade-offs Summary

### Comparison Matrix

| Approach | Isolation | Performance | DX | Complexity | Best For |
|----------|-----------|-------------|----|-----------|-----------| 
| **Module Federation** | Medium | Good | Good | Medium | Same framework teams |
| **iframe** | Perfect | Poor | Poor | Low | 3rd-party content |
| **Web Components** | Good | Excellent | Medium | Medium | Cross-framework |

---

### Detailed Trade-offs

#### Module Federation
```
✅ Pros:
- Share dependencies (smaller bundles)
- Good DX (same framework)
- Dynamic loading
- True independent deployment

❌ Cons:
- Webpack 5 only (vendor lock)
- Complex debugging
- Version coordination needed
- Runtime overhead

Use: Large orgs, same framework
```

---

#### iframe
```
✅ Pros:
- Perfect isolation (CSS, JS, security)
- Technology agnostic
- Simple concept
- Security sandbox

❌ Cons:
- Poor UX (scrolling, routing, layout)
- High memory/performance cost
- SEO bad
- Communication overhead

Use: 3rd-party embed, legacy migration
```

---

#### Web Components
```
✅ Pros:
- Framework agnostic
- Standards-based (future-proof)
- Good encapsulation
- Native performance

❌ Cons:
- Poor DX (vanilla)
- Missing framework features
- SSR hard
- Polyfills needed (old browsers)

Use: Design systems, cross-framework apps
```

---

## Real-world Patterns

### Pattern 1: Shell + Module Federation (Most common)

```
Architecture:
┌──────────────────────────────┐
│   Shell App (Host)           │
│   - Routing                  │
│   - Layout (Header, Nav)     │
│   - Auth                     │
│   - Load remotes             │
└──────────────────────────────┘
         ↓ (Module Federation)
┌─────────┬─────────┬──────────┐
│ Products│  Cart   │ Checkout │
│  Remote │ Remote  │  Remote  │
└─────────┴─────────┴──────────┘

Teams: 3-5 teams, React/Vue/Angular
Scale: Medium-Large (10-50 devs)
```

**Key decisions**:
- Shell owns: Routing, auth, common layout
- Remotes own: Feature logic, UI
- Shared: Design system, utilities (via Module Federation)

---

### Pattern 2: iframe + postMessage (Legacy integration)

```
Architecture:
┌──────────────────────────────┐
│   Modern App (React)         │
│   ┌────────────────────────┐ │
│   │ iframe: Legacy (ASP.NET│ │
│   │ /admin)                │ │
│   └────────────────────────┘ │
└──────────────────────────────┘

Use case: Gradual migration
- New features: React
- Old features: iframe legacy
- Communicate: postMessage
```

---

### Pattern 3: Monorepo + Build-time composition

```
monorepo/
├── packages/
│   ├──
shell/
│   ├── products/
│   ├── cart/
│   └── shared/
└── apps/
    └── main/ (compose packages)

Build: Lerna, Nx, Turborepo
Deploy: Single artifact (not micro-frontends technically)

Benefits:
- Code sharing easy
- Monolithic deployment (simpler)
- No runtime composition overhead

Trade-off:
- Not true independent deployment
- Still better than pure monolith (clear boundaries)
```

---

## Key Takeaways

### Micro-frontends
- **Purpose**: Organizational scaling, team autonomy
- **Cost**: Complexity, performance overhead
- **Use**: Large teams (50+ devs), multiple domains
- **Don't use**: Small teams, simple apps

### Module Federation
- **Best for**: Modern stacks, same framework
- **Pros**: Code sharing, dynamic loading
- **Cons**: Webpack lock-in, version coordination

### iframe
- **Best for**: 3rd-party content, security isolation
- **Pros**: Perfect isolation, simple
- **Cons**: Poor UX, performance

### Web Components
- **Best for**: Cross-framework, design systems
- **Pros**: Standards, framework agnostic
- **Cons**: Poor DX, missing features

---

**Final Advice**:
> Micro-frontends solve **organizational problems**, not technical problems. If team < 20 devs, monolith với clear modules đủ. Chỉ adopt micro-frontends khi pain points rõ ràng: merge conflicts daily, deploy bottlenecks, team stepping on each other. Start simple, evolve khi cần.