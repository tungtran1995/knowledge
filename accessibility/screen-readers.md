# Screen Readers – How Blind Users Navigate the Web

> **Goal:** Understand how screen readers work and how to test with them.

---

## 1. How Screen Readers Work (High Level)

### The Basics

**Screen reader = Software that reads web content aloud**

**Popular screen readers:**
- **JAWS** (Windows) - Most popular, expensive ($1000+)
- **NVDA** (Windows) - Free, popular
- **VoiceOver** (macOS/iOS) - Built-in, free
- **TalkBack** (Android) - Built-in, free
- **ORCA** (Linux) - Free

---

### What They Read

**1. Element type**
```html
<button>Save</button>
→ "Button, Save"

<a href="/about">About</a>
→ "Link, About"

<h1>Welcome</h1>
→ "Heading level 1, Welcome"
```

**2. State**
```html
<button aria-pressed="true">Bold</button>
→ "Button, Bold, pressed"

<input type="checkbox" checked />
→ "Checkbox, checked"
```

**3. Properties**
```html
<button aria-label="Close dialog">×</button>
→ "Button, Close dialog"

<input aria-invalid="true" aria-errormessage="error1" />
→ "Invalid, see error message"
```

---

### Navigation Methods

**1. Tab key** - Jump between focusable elements
**2. Headings** - Jump between h1, h2, h3, etc.
**3. Landmarks** - Jump between `<nav>`, `<main>`, `<aside>`, etc.
**4. Links** - List all links on page
**5. Forms** - List all form elements
**6. Virtual cursor** - Read line by line (arrow keys)

---

## 2. Testing with Screen Readers

### VoiceOver (macOS/iOS)

**How to start:**
- macOS: Cmd+F5
- iOS: Settings → Accessibility → VoiceOver → On

**Basic commands (macOS):**
```
VO = Ctrl+Option (shorthand)

VO+A: Start reading
VO+←/→: Previous/next item
VO+Cmd+H: Next heading
VO+U: Rotor (navigate by headings, links, etc.)
Ctrl: Stop reading
```

**Testing workflow:**
1. Start VoiceOver (Cmd+F5)
2. Navigate to your page
3. VO+A to start reading
4. VO+U to open rotor → Select "Headings"
5. Verify structure makes sense

---

### NVDA (Windows - Free)

**Download:** https://www.nvaccess.org

**Basic commands:**
```
Ctrl: Stop reading
Insert+↓: Say all (read entire page)
H: Next heading
K: Next link
F: Next form field
B: Next button
Insert+F7: Elements list
```

**Testing workflow:**
1. Start NVDA (Ctrl+Alt+N)
2. Navigate to page
3. Press H to jump through headings
4. Press Insert+F7 to see all headings/links
5. Verify logical order

---

### Quick Test (No Screen Reader)

**Browser tools:**
- Chrome: Accessibility tree (DevTools → Accessibility pane)
- Firefox: Accessibility Inspector
- Edge: Same as Chrome

**What to check:**
- Element names (is button labeled?)
- Element roles (is div role="button"?)
- State (is checkbox checked?)

---

## 3. Alt Text for Images

### The Rules

**Rule 1:** Informative images need alt text
```html
<!-- ✅ Describes what's in image -->
<img src="chart.png" alt="Sales increased 50% in Q3" />

<!-- ❌ Useless alt -->
<img src="chart.png" alt="Chart" />
```

**Rule 2:** Decorative images get empty alt
```html
<!-- ✅ Decorative border -->
<img src="border.png" alt="" />

<!-- or better: CSS background -->
<div style="background-image: url(border.png)"></div>
```

**Rule 3:** Functional images describe action
```html
<!-- ✅ Button image -->
<button>
  <img src="delete.svg" alt="Delete item" />
</button>

<!-- or better: aria-label on button -->
<button aria-label="Delete item">
  <img src="delete.svg" alt="" />
</button>
```

---

### Writing Good Alt Text

**Guidelines:**
- Concise (< 150 characters)
- Describe content, not "image of"
- Don't  repeat nearby text
- Include text if image contains text

```html
<!-- ❌ Bad -->
<img alt="Image of a dog" />

<!-- ✅ Good -->
<img alt="Golden retriever puppy playing with ball" />

<!-- ❌ Redundant -->
<figure>
  <img alt="Golden retriever puppy" />
  <figcaption>Golden retriever puppy</figcaption>
</figure>

<!-- ✅ Better -->
<figure>
  <img alt="" /> <!-- Empty, caption describes -->
  <figcaption>Golden retriever puppy playing</figcaption>
</figure>
```

---

### Complex Images

**Charts, diagrams:**
```html
<!-- Long description in adjacent text -->
<img
  src="sales-chart.png"
  alt="Sales chart showing 50% increase"
  aria-describedby="chart-description"
/>
<div id="chart-description">
  Our sales increased from $100k in Q1 to $150k in Q3,
  representing a 50% growth over the period.
</div>
```

---

## 4. Form Labels

### Label Association

```html
<!-- ✅ Explicit association (preferred) -->
<label for="email">Email</label>
<input type="email" id="email" name="email" />

<!-- ✅ Implicit association (also works) -->
<label>
  Email
  <input type="email" name="email" />
</label>

<!-- ❌ No association -->
<label>Email</label>
<input type="email" /> <!-- SR doesn't know label -->
```

**Screen reader announces:** "Email, edit text" (with label), "Edit text" (without).

---

### Placeholder ≠ Label

```html
<!-- ❌ Only placeholder (insufficient) -->
<input type="email" placeholder="Enter your email" />

<!-- ✅ Both label and placeholder -->
<label for="email">Email</label>
<input
  type="email"
  id="email"
  placeholder="e.g., john@example.com"
/>
```

**Why:** Placeholder disappears when typing, screen reader may not read it.

---

### Required Fields

```html
<!-- ✅ Multiple indicators -->
<label for="email">
  Email <span aria-label="required">*</span>
</label>
<input type="email" id="email" required />

<!-- ✅ Or visible "required" text -->
<label for="email">Email (required)</label>
<input type="email" id="email" required aria-required="true" />
```

---

### Error Messages

```html
<label for="email">Email</label>
<input
  type="email"
  id="email"
  aria-invalid="true"
  aria-describedby="email-error"
/>
<span id="email-error" role="alert">
  Please enter a valid email address
</span>
```

**Screen reader announces:** "Email, invalid, see error: Please enter a valid email address"

---

## 5. Common Screen Reader Issues

### Issue 1: Images without alt

```html
<!-- ❌ SR reads filename -->
<img src="profile_photo_user_123.png" />
→ "Image, profile underscore photo underscore user underscore 123"

<!-- ✅ Descriptive alt -->
<img src="profile_photo_user_123.png" alt="User profile photo" />
→ "Image, User profile photo"
```

---

### Issue 2: Empty links

```html
<!-- ❌ SR reads URL -->
<a href="/settings">
  <svg>...</svg>
</a>
→ "Link, slash settings"

<!-- ✅ Accessible name -->
<a href="/settings" aria-label="Settings">
  <svg aria-hidden="true">...</svg>
</a>
→ "Link, Settings"
```

---

### Issue 3: Buttons with only icons

```html
<!-- ❌ No accessible name -->
<button>
  <svg>...</svg>
</button>
→ "Button" (what does it do?)

<!-- ✅ aria-label -->
<button aria-label="Delete item">
  <svg aria-hidden="true">...</svg>
</button>
→ "Button, Delete item"
```

---

### Issue 4: Fake buttons

```html
<!-- ❌ Not announced as button -->
<div onClick={...}>Click me</div>
→ "Click me" (just text, not interactive)

<!-- ✅ Real button -->
<button onClick={...}>Click me</button>
→ "Button, Click me"
```

---

## 6. Testing Checklist

### Before Launch

**Images:**
- ☐ All images have alt text (or alt="" if decorative)
- ☐ Alt text describes content, not "image of"
- ☐ Complex images have long descriptions

**Forms:**
- ☐ All inputs have labels
- ☐ Labels associated with inputs (id/for)
- ☐ Required fields indicated
- ☐ Error messages linked (aria-describedby)

**Interactive elements:**
- ☐ All buttons/links have accessible names
- ☐ Icon buttons have aria-label
- ☐ Custom controls have roles

**Structure:**
- ☐ Heading hierarchy correct (h1 → h2 → h3)
- ☐ Landmarks used (main, nav, aside)
- ☐ Link text descriptive (not "click here")

---

### Screen Reader Test (15 minutes)

**Quick test flow:**
1. Start screen reader (VoiceOver/NVDA)
2. Navigate by headings (H key or VO+Cmd+H)
   - ☐ Headings make sense out of context?
3. Navigate by links (K key or rotor)
   - ☐ Link text descriptive?
4. Navigate by forms (F key)
   - ☐ All inputs labeled?
5. Tab through page
   - ☐ All interactive elements reachable?
   - ☐ Announced correctly?

---

## 7. Common Mistakes

### ❌ Mistake 1: Assuming visual = accessible

```
Just because it looks good ≠ screen reader accessible
- Visual "required" asterisk not read
- Color-only error states not announced
- Modal visible but SR doesn't know
```

---

### ❌ Mistake 2: Testing only with mouse

```
Mouse works ≠ SR works
- SR uses keyboard, not mouse
- SR reads semantic HTML, not visual CSS
```

---

### ❌ Mistake 3: Placeholder as label

```html
<!-- Users see placeholder, SR may not read it -->
<input placeholder="Search..." />
```

---

### ❌ Mistake 4: Icon-only buttons

```html
<!-- Looks obvious visually, SR hears nothing -->
<button><DeleteIcon /></button>
```

---

## Tổng Kết

**How to support screen reader users:**

1. **Semantic HTML**
   - Use correct elements (button, heading, etc.)
   - Free accessibility

2. **All Images Have Alt**
   - Informative → descriptive alt
   - Decorative → empty alt

3. **All Inputs Have Labels**
   - Use `<label for="...">`
   - Not just placeholders

4. **Test With Actual SR**
   - 15-minute test reveals most issues
   - Try VoiceOver (free on Mac) or NVDA (free on Windows)

**Remember:** If you can't complete your task with eyes closed + screen reader, users can't either.

---

**Next:** [Testing Tools →](./testing.md)
