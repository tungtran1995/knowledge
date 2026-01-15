# ARIA Guide – When Semantic HTML Isn't Enough

> **Golden Rule:** No ARIA is better than bad ARIA.

---

## 1. What is ARIA?

**Accessible Rich Internet Applications**

### Purpose

Fill gaps when semantic HTML can't express something:
- Custom components (tabs, accordions, dialogs)
- Dynamic content (live regions)
- Complex widgets (tree views, comboboxes)

---

### The Rules

**Rule 1:** Use semantic HTML first
```html
<!-- ❌ Don't use ARIA if HTML works -->
<div role="button" tabindex="0" onclick="...">Click</div>

<!-- ✅ HTML button is better -->
<button onclick="...">Click</button>
```

**Rule 2:** Don't change native semantics
```html
<!-- ❌ Never do this -->
<button role="heading">Not a button anymore</button>

<!-- ✅ Use correct element -->
<h2>This is a heading</h2>
```

**Rule 3:** Keyboard support required
```html
<!-- ❌ ARIA without keyboard = broken -->
<div role="button">Click</div>

<!-- ✅ Add keyboard support -->
<div role="button" tabindex="0" onKeyPress={handleKey}>Click</div>
```

**Rule 4:** Don't hide focusable elements
```html
<!-- ❌ Keyboard trap -->
<button aria-hidden="true">I can still be focused!</button>

<!-- ✅ Hide from everyone if decorative -->
<button aria-hidden="true" tabindex="-1">Truly hidden</button>
```

---

## 2. When to Use ARIA

### ✅ Use ARIA When:

**1. Custom components not in HTML**
```
- Tabs
- Accordions
- Modals/Dialogs
- Tooltips
- Dropdown menus
- Comboboxes
```

**2. Dynamic content**
```
- Live regions (notifications, alerts)
- Loading states
- Error messages
```

**3. Complex widgets**
```
- Tree views
- Data grids
- Carousels
```

---

### ❌ DON'T Use ARIA When:

**1. Semantic HTML exists**
```html
<!-- ❌ -->
<div role="button">Click</div>

<!-- ✅ -->
<button>Click</button>
```

**2. Native behavior is better**
```html
<!-- ❌ Custom dropdown with 50 lines of ARIA -->
<div role="listbox">...</div>

<!-- ✅ Native select -->
<select>
  <option>Option 1</option>
</select>
```

---

## 3. Common ARIA Roles

### Landmark Roles (Redundant with HTML5)

```html
<!-- Old way (ARIA) -->
<div role="navigation">...</div>
<div role="main">...</div>

<!-- ✅ Modern way (HTML5) -->
<nav>...</nav>
<main>...</main>
```

**Note:** HTML5 landmarks preferred. ARIA roles for legacy support only.

---

### Widget Roles (Common)

**`role="button"`**
```html
<div role="button" <tabindex="0" onClick={...} onKeyPress={...}>
  Custom Button
</div>
```
Use when: Styling limitations force using div instead of `<button>`.

---

**`role="tab"`, `"tablist"`, `"tabpanel"`**
```html
<div role="tablist">
  <button role="tab" aria-selected="true" aria-controls="panel1">
    Tab 1
  </button>
  <button role="tab" aria-selected="false" aria-controls="panel2">
    Tab 2
  </button>
</div>

<div role="tabpanel" id="panel1">Panel 1 content</div>
<div role="tabpanel" id="panel2" hidden>Panel 2 content</div>
```

---

**`role="dialog"`**
```html
<div role="dialog" aria-labelledby="dialog-title" aria-modal="true">
  <h2 id="dialog-title">Confirm Delete</h2>
  <p>Are you sure?</p>
  <button>Cancel</button>
  <button>Delete</button>
</div>
```

---

## 4. ARIA Properties

### aria-label

**Use:** Provide accessible name when no visible text.

```html
<!-- Icon button without text -->
<button aria-label="Close">
  <svg>...</svg>
</button>

<!-- Search input (placeholder not enough) -->
<input type="text" placeholder="Search" aria-label="Search posts" />
```

---

### aria-labelledby

**Use:** Reference existing elements as label.

```html
<section aria-labelledby="features-heading">
  <h2 id="features-heading">Features</h2>
  <p>Our amazing features...</p>
</section>

<!-- Modal with title -->
<div role="dialog" aria-labelledby="modal-title">
  <h2 id="modal-title">Delete Confirmation</h2>
  ...
</div>
```

---

### aria-describedby

**Use:** Additional description/help text.

```html
<input
  type="password"
  aria-labelledby="password-label"
  aria-describedby="password-hint"
/>
<label id="password-label">Password</label>
<p id="password-hint">Must be at least 8 characters</p>
```

---

### aria-hidden

**Use:** Hide decorative elements from screen readers.

```html
<!-- Decorative icon (text explains already) -->
<button>
  <svg aria-hidden="true">...</svg>
  Delete
</button>

<!-- Duplicate content -->
<button aria-label="Close dialog">
  <span aria-hidden="true">×</span>
</button>
```

**⚠️ Warning:** Don't hide meaningful content!

---

### aria-live

**Use:** Announce dynamic content changes.

**Values:**
- `off` (default): Don't announce
- `polite`: Announce when user idle
- `assertive`: Announce immediately (use sparingly)

```html
<!-- Success notification -->
<div role="status" aria-live="polite">
  Item added to cart
</div>

<!-- Error (urgent) -->
<div role="alert" aria-live="assertive">
  Payment failed. Please try again.
</div>
```

**Shorthand roles:**
- `role="status"` = `aria-live="polite"`
- `role="alert"` = `aria-live="assertive"`

---

### aria-invalid & aria-errormessage

**Use:** Form validation.

```html
<label for="email">Email</label>
<input
  type="email"
  id="email"
  aria-invalid="true"
  aria-errormessage="email-error"
/>
<span id="email-error" role="alert">
  Please enter a valid email
</span>
```

---

### aria-expanded

**Use:** Collapsible sections (accordions, dropdowns).

```html
<button
  aria-expanded="false"
  aria-controls="menu"
  onClick={() => setOpen(!open)}
>
  Menu
</button>

<div id="menu" hidden={!open}>
  Menu items...
</div>
```

**Values:** `true` | `false`

---

### aria-current

**Use:** Current item in navigation.

```html
<nav>
  <a href="/" aria-current="page">Home</a>
  <a href="/about">About</a>
  <a href="/contact">Contact</a>
</nav>
```

**Values:** `page` | `step` | `location` | `date` | `time` | `true` | `false`

---

## 5. Common Patterns

### Modal Dialog

```html
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
>
  <h2 id="dialog-title">Title</h2>
  <div>Content...</div>
  <button onClick={closeModal}>Close</button>
</div>
```

**Required:**
- `role="dialog"`
- `aria-labelledby` or `aria-label`
- Focus management (trap focus inside)
- Keyboard: Esc to close

---

### Accordion

```html
<div>
  <h2>
    <button
      aria-expanded={isOpen}
      aria-controls="section1"
      onClick={toggle}
    >
      Section 1
    </button>
  </h2>
  <div id="section1" hidden={!isOpen}>
    Content...
  </div>
</div>
```

---

### Loading State

```html
<!-- Before loading -->
<button>Load More</button>

<!-- During loading -->
<button aria-busy="true" aria-live="polite">
  Loading...
</button>

<!-- After loading -->
<div role="status" aria-live="polite">
  10 more items loaded
</div>
```

---

### Custom Checkbox

```html
<div
  role="checkbox"
  aria-checked={checked}
  aria-labelledby="checkbox-label"
  tabIndex={0}
  onClick={toggle}
  onKeyPress={handleKey}
>
  <span id="checkbox-label">Agree to terms</span>

</div>
```

**Note:** Prefer `<input type="checkbox">` when possible!

---

## 6. ARIA Anti-Patterns

### ❌ Don't: Override native semantics

```html
<!-- ❌ Breaking native button -->
<button role="heading">I'm confused</button>

<!-- ✅ Use correct element -->
<h2>I'm a heading</h2>
```

---

### ❌ Don't: ARIA spam

```html
<!-- ❌ Redundant ARIA -->
<button role="button" aria-label="Submit button">
  Submit
</button>

<!-- ✅ Native semantics sufficient -->
<button>Submit</button>
```

---

### ❌ Don't: Hide important content

```html
<!-- ❌ Hiding actual content -->
<main aria-hidden="true">
  <!-- Important stuff hidden from SR -->
</main>

<!-- ✅ Only hide decorative -->
<button>
  Delete
  <svg aria-hidden="true">...</svg>
</button>
```

---

### ❌ Don't: Forget keyboard support

```html
<!-- ❌ ARIA without keyboard -->
<div role="button" onClick={...}>Click</div>

<!-- ✅ Full keyboard support -->
<div
  role="button"
  tabIndex={0}
  onClick={...}
  onKeyPress={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      ...
    }
  }}
>
  Click
</div>
```

---

## 7. Testing ARIA

**Tools:**
- Chrome DevTools → Accessibility tree
- axe DevTools → Flags ARIA errors
- Screen reader → Verify announcements

**Manual checks:**
- ☐ Role announced correctly?
- ☐ State changes announced (aria-expanded, aria-checked)?
- ☐ Live regions announce updates?
- ☐ Relationships clear (aria-labelledby, aria-controls)?

---

## Tổng Kết

**ARIA Hierarchy:**
```
1. Semantic HTML (best)
   ↓ (if not sufficient)
2. ARIA (when needed)
   ↓ (if too complex)
3. Consider simpler design
```

**Key Principles:**

1. **HTML First**
   - 90% of cases covered by semantic HTML
   - ARIA for remaining 10%

2. **Don't Break Native**
   - Never override native semantics
   - Test with keyboard + screen reader

3. **Test Thoroughly**
   - ARIA errors worse than no ARIA
   - Automated tools + manual testing

**When in doubt:** Use semantic HTML, not ARIA.

---

**Next:** [Keyboard Navigation →](./keyboard-nav.md)
