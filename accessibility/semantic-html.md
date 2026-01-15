# Semantic HTML – Foundation of Accessibility

> **Core principle:** Use the right element for the job.

---

## 1. Why Semantic HTML Matters

### The Problem with Divs

```html
<!-- ❌ All divs + CSS = Looks right, works wrong -->
<div class="button" onclick="submit()">Submit</div>
<div class="header">Welcome</div>
<div class="nav">Menu</div>
```

**What's wrong:**
- Screen reader: "clickable, Submit" (not announced as button)
- Keyboard: No tab navigation
- Browser: No default behaviors (Enter key, focus, etc.)

```html
<!-- ✅ Semantic = Looks right + works right -->
<button type="submit">Submit</button>
<h1>Welcome</h1>
<nav>Menu</nav>
```

**What's right:**
- Screen reader: "Button, Submit"
- Keyboard: Tab to focus, Enter/Space to click
- Browser: All default behaviors included

---

## 2. Proper Element Usage

### Buttons vs Links vs Divs

**Rule:**
```
<button> → Does something (submit, toggle, open)
<a> → Goes somewhere (navigation, links)
<div> → Container only (no interaction)
```

**Examples:**

```html
<!-- ✅ Correct -->
<button onClick={handleSubmit}>Save</button>
<button onClick={toggleMenu}>Menu</button>
<a href="/about">About Us</a>

<!-- ❌ Wrong -->
<div onClick={handleSubmit}>Save</div> <!-- Should be <button> -->
<button onClick={() => navigate('/about')}>About</button> <!-- Should be <a> -->
<a href="#" onClick={toggleMenu}>Menu</a> <!-- Should be <button> with no href -->
```

---

### Lists

**When to use:**
- Navigation menus
- Sets of related items
- Steps in a process

```html
<!-- ✅ Good -->
<nav>
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<!-- ❌ Bad -->
<nav>
  <a href="/">Home</a>
  <a href="/about">About</a>
</nav>
```

**Why better:** Screen reader announces "List, 2 items" → User knows what to expect.

---

### Forms

```html
<!-- ✅ Correct form structure -->
<form>
  <label for="email">Email</label>
  <input type="email" id="email" name="email" required />
  
  <label for="password">Password</label>
  <input type="password" id="password" name="password" required />
  
  <button type="submit">Log in</button>
</form>

<!-- ❌ Missing labels -->
<form>
  <input type="email" placeholder="Email" />
  <input type="password" placeholder="Password" />
  <div onclick="submit()">Log in</div>
</form>
```

**Critical:** `<label>` + `for` attribute connects label to input.

---

## 3. Heading Hierarchy

### The Rules

```
h1 → Page title (one per page)
 ├─ h2 → Main sections
 │   ├─ h3 → Subsections
 │   │   └─ h4 → Sub-subsections
 │   │       └─ h5, h6 → Rarely needed
```

---

### ✅ Correct Hierarchy

```html
<h1>Dashboard</h1>

<h2>Recent Activity</h2>
<p>Your recent actions...</p>

<h2>Projects</h2>

<h3>Project Alpha</h3>
<p>Details about Alpha...</p>

<h3>Project Beta</h3>
<p>Details about Beta...</p>

<h2>Settings</h2>
<h3>Account Settings</h3>
<h3>Privacy Settings</h3>
```

---

### ❌ Common Mistakes

```html
<!-- ❌ Skipping levels -->
<h1>Dashboard</h1>
<h4>Recent Activity</h4> <!-- Skipped h2, h3 -->

<!-- ❌ Using for styling -->
<h3 class="small-text">Small heading</h3>
<!-- Instead: Use CSS to style h2 smaller -->

<!-- ❌ Multiple h1s -->
<h1>Dashboard</h1>
<h1>Recent Activity</h1> <!-- Should be h2 -->
```

---

### Why It Matters

**Screen reader users:**
- Use headings to navigate (skip to sections)
- Expect hierarchy: h1 → h2 → h3
- Broken hierarchy = confusing

**Testing:** Use browser extension to view heading outline.

---

## 4. Landmarks (HTML5 Regions)

### Main Landmarks

```html
<header>    <!-- Site/page header -->
<nav>       <!-- Navigation -->
<main>      <!-- Main content (one per page) -->
<aside>     <!-- Sidebar, tangential content -->
<footer>    <!-- Site/page footer -->
<article>   <!-- Independent content (blog post, comment) -->
<section>   <!-- Thematic grouping -->
```

---

### Proper Usage

```html
<!DOCTYPE html>
<html>
<body>
  <header>
    <nav>
      <ul>
        <li><a href="/">Home</a></li>
      </ul>
    </nav>
  </header>
  
  <main>
    <h1>Page Title</h1>
    
    <article>
      <h2>Article Title</h2>
      <p>Content...</p>
    </article>
    
    <aside>
      <h2>Related Links</h2>
      <ul>...</ul>
    </aside>
  </main>
  
  <footer>
    <p>&copy; 2024 Company</p>
  </footer>
</body>
</html>
```

---

### Multiple Landmarks (Need Labels)

```html
<!-- ✅ Multiple navs with labels -->
<nav aria-label="Main navigation">
  <ul>...</ul>
</nav>

<nav aria-label="Footer navigation">
  <ul>...</ul>
</nav>

<!-- ✅ Multiple sections with headings -->
<section>
  <h2>Features</h2>
  ...
</section>

<section>
  <h2>Pricing</h2>
  ...
</section>
```

**Why:** Screen reader user can jump to "Main navigation" or "Footer navigation".

---

## 5. Tables

### When to Use Tables

✅ **Use for tabular data:**
- Spreadsheets
- Comparison charts
- Data grids

❌ **DON'T use for layout:**
- Page structure
- Grids of cards

---

### Accessible Table Structure

```html
<table>
  <caption>Monthly Sales Report</caption>
  
  <thead>
    <tr>
      <th scope="col">Month</th>
      <th scope="col">Revenue</th>
      <th scope="col">Profit</th>
    </tr>
  </thead>
  
  <tbody>
    <tr>
      <th scope="row">January</th>
      <td>$10,000</td>
      <td>$2,000</td>
    </tr>
    <tr>
      <th scope="row">February</th>
      <td>$12,000</td>
      <td>$2,500</td>
    </tr>
  </tbody>
</table>
```

**Key elements:**
- `<caption>`: Table title
- `<th scope="col">`: Column header
- `<th scope="row">`: Row header
- `<thead>`, `<tbody>`: Structure

**Screen reader:** "January, Revenue: $10,000" (associates headers with data)

---

## 6. Common Patterns

### Skip Links

```html
<body>
  <a href="#main" class="skip-link">Skip to main content</a>
  
  <header>...</header>
  <nav>...</nav>
  
  <main id="main">
    <!-- Main content -->
  </main>
</body>

<style>
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  z-index: 1000;
}

.skip-link:focus {
  top: 0;
}
</style>
```

**Why:** Keyboard users can skip repetitive navigation.

---

### Image Buttons

```html
<!-- ❌ Wrong -->
<button>
  <img src="close.svg" />
</button>

<!-- ✅ Right -->
<button aria-label="Close">
  <img src="close.svg" alt="" />
</button>

<!-- or -->
<button>
  <img src="close.svg" alt="Close" />
</button>
```

**Rule:** If image is the only content, button needs label.

---

### Icon Buttons

```html
<!-- ✅ Accessible icon button -->
<button aria-label="Delete item">
  <svg>...</svg> <!-- Icon -->
</button>

<!-- or with visible text -->
<button>
  <svg aria-hidden="true">...</svg>
  <span>Delete</span>
</button>
```

---

## 7. Checklist

### Before Shipping

**Elements:**
- ☐ No `<div>` or `<span>` with click handlers (use `<button>`)
- ☐ Links go somewhere (`<a href>`), buttons do something (`<button>`)
- ☐ All form inputs have labels
- ☐ Headings in proper order (h1 → h2 → h3)
- ☐ Only one `<main>` per page
- ☐ Landmarks used (`<header>`, `<nav>`, `<main>`, `<footer>`)

**Lists:**
- ☐ Navigation menus use `<ul>` or `<ol>`
- ☐ Related items use lists

**Tables:**
- ☐ Tables have `<caption>`
- ☐ Headers have `scope` attribute
- ☐ `<thead>` and `<tbody>` used

---

## Tổng Kết

**Key Principles:**

1. **Right Element = Free Accessibility**
   - `<button>` → Free keyboard support
   - `<a>` → Free navigation
   - `<label>` → Free association

2. **Structure Matters**
   - Headings create outline
   - Landmarks enable navigation
   - Lists group related items

3. **Don't DIY Browser Features**
   - Use native elements (button, input, select)
   - Browser handles accessibility
   - Custom = more work + more bugs

---

**Next:** [ARIA Guide →](./aria-guide.md) (when semantic HTML isn't enough)
