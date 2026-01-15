# Keyboard Navigation – Essential for Accessibility

> **Key fact:** 15-20% of users rely on keyboard navigation.

---

## 1. Why Keyboard Navigation Matters

### Who Uses Keyboard-Only?

**Motor disabilities:**
- Cannot use mouse
- Use specialized input devices
- Keyboard = primary interface

**Power users:**
- Faster than mouse
- Vi

m/Emacs users
- Developers, writers

**Temporary situations:**
- Broken mouse/trackpad
- Using external keyboard with tablet
- Arthritic hands (mouse painful)

---

## 2. Tab Order

### Natural Tab Order

**Rule:** DOM order = Tab order

```html
<!-- ✅ Tab order: 1 → 2 → 3 -->
<button>First</button>
<a href="#">Second</a>
<input type="text" />
```

**What's tabbable by default:**
- `<a href>`
- `<button>`
- `<input>`, `<select>`, `<textarea>`
- `<iframe>`
- Elements with `tabindex="0"` or positive tabindex

---

### tabindex Attribute

**Values:**

**`tabindex="-1"`** → Focusable by JavaScript, not Tab key
```html
<div tabindex="-1" ref={errorRef}>
  Error message
</div>

<!-- Focus programmatically -->
errorRef.current.focus();
```

**`tabindex="0"`** → Add to tab order (natural position)
```html
<div role="button" tabindex="0" onClick={...}>
  Custom button
</div>
```

**`tabindex="1+"` (positive)** → ❌ **AVOID**
```html
<!-- ❌ DON'T: Breaks natural order -->
<button tabindex="3">First in DOM, third in tab</button>
<input tabindex="1" /> <!-- Second in DOM, first in tab -->
<input tabindex="2" /> <!-- Third in DOM, second in tab -->
```

**Why avoid:** Confusing, hard to maintain, breaks expectations.

---

### ✅ Good Tab Order Example

```html
<form>
  <!-- 1 --> <input type="text" placeholder="Name" />
  <!-- 2 --> <input type="email" placeholder="Email" />
  <!-- 3 --> <button>Submit</button>
  <!-- 4 --> <a href="/cancel">Cancel</a>
</form>
```

Tab order matches visual order = good UX.

---

### ❌ Bad Tab Order Example

```html
<!-- Visual order: Submit, Name, Email -->
<button style="order: 1">Submit</button>
<input style="order: 2" />
<input style="order: 3" />

<!-- But tab order follows DOM: Submit → input 1 → input 2 -->
```

**Fix:** Match DOM order to visual order, or use `tabindex` cautiously.

---

## 3. Focus Management

### What to Focus

**Rule:** All interactive elements should be focusable.

```html
<!-- ✅ Focusable (native) -->
<button>Click</button>
<a href="#">Link</a>
<input />

<!-- ❌ Not focusable -->
<div onClick={...}>Click</div>

<!-- ✅ Made focusable -->
<div role="button" tabindex="0" onClick={...}>Click</div>
```

---

### Focus Visible (CSS)

**Problem:** Default focus outline sometimes ugly.

```css
/* ❌ NEVER remove outline without replacement */
button:focus {
  outline: none; /* Keyboard users lost! */
}

/* ✅ Custom focus style */
button:focus-visible {
  outline: 2px solid blue;
  outline-offset: 2px;
}
```

**`:focus-visible`** → Only shows when keyboard-focused (not mouse-clicked).

---

### Focus Trap (Modals)

**Concept:** Keep focus inside modal until closed.

**Pattern:**
1. Open modal → Move focus to first element
2. Tab from last → Loop to first
3. Shift+Tab from first → Loop to last
4. Close modal → Return focus to trigger

```javascript
function Modal({ isOpen, onClose }) {
  const firstRef = useRef();
  const lastRef = useRef();
  
  useEffect(() => {
    if (isOpen) {
      firstRef.current?.focus();
    }
  }, [isOpen]);
  
  const handleKeyDown = (e) => {
    if (e.key === 'Escape') {
      onClose();
    }
    
    // Trap focus
    if (e.key === 'Tab') {
      if (e.shiftKey && document.activeElement === firstRef.current) {
        e.preventDefault();
        lastRef.current.focus();
      } else if (!e.shiftKey && document.activeElement === lastRef.current) {
        e.preventDefault();
        firstRef.current.focus();
      }
    }
  };
  
  return (
    <div role="dialog" onKeyDown={handleKeyDown}>
      <button ref={firstRef} onClick={onClose}>Close</button>
      <div>Content...</div>
      <button ref={lastRef}>Save</button>
    </div>
  );
}
```

**Libraries:** `focus-trap-react`, `react-focus-lock`

---

### Programmatic Focus

**When to move focus:**
- Modal opens → Focus first element
- Error occurs → Focus error message
- Item deleted → Focus next item

```html
<button onClick={handleDelete}>Delete</button>

{error && (
  <div role="alert" tabindex="-1" ref={errorRef}>
    {error}
  </div>
)}
```

```javascript
function handleDelete() {
  try {
    deleteItem();
  } catch (err) {
    setError(err.message);
    errorRef.current?.focus(); // Focus error
  }
}
```

---

## 4. Keyboard Shortcuts

### Standard Shortcuts

**Browser/OS level** (don't override):
- Tab: Next focusable
- Shift+Tab: Previous focusable
- Enter: Activate button/link
- Space: Activate button, toggle checkbox
- Arrow keys: Radio buttons, select options
- Escape: Close modal/dialog
- Ctrl+F: Find

---

### Custom Shortcuts

**When to add:**
- Rich applications (editors, dashboards)
- Improve power user experience
- NOT for simple sites

**Best practices:**
- Document shortcuts (help tooltip)
- Allow users to disable
- Avoid conflicts with browser/screen readers
- Use modifier keys (Ctrl, Alt, Shift)

```javascript
function Editor() {
  useEffect(() => {
    const handleKeyDown = (e) => {
      // Ctrl+S to save
      if (e.ctrlKey && e.key === 's') {
        e.preventDefault();
        save();
      }
      
      // Ctrl+B to bold
      if (e.ctrlKey && e.key === 'b') {
        e.preventDefault();
        toggleBold();
      }
    };
    
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, []);
}
```

---

## 5. Skip Links

### Purpose

Allow keyboard users to skip repetitive content (navigation, headers).

**Problem:**
```
User tabs through site:
Tab 1-20: Navigation menu
Tab 21-30: Sidebar
Tab 31: First content link

Frustrating!
```

**Solution: Skip link**

```html
<body>
  <a href="#main" class="skip-link">Skip to main content</a>
  
  <header>...</header>
  <nav>
    <!-- 50 links -->
  </nav>
  
  <main id="main">
    <h1>Content starts here</h1>
  </main>
</body>
```

```css
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
```

**Behavior:**
- Hidden by default
- Visible on focus (first Tab)
- Clicking jumps to main content

---

## 6. Common Patterns

### Dropdown Menu

```javascript
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  
  const handleKeyDown = (e) => {
    switch (e.key) {
      case 'Enter':
      case ' ':
        e.preventDefault();
        setIsOpen(!isOpen);
        break;
      case 'Escape':
        setIsOpen(false);
        break;
      case 'ArrowDown':
        e.preventDefault();
        // Focus next item
        break;
      case 'ArrowUp':
        e.preventDefault();
        // Focus previous item
        break;
    }
  };
  
  return (
    <div>
      <button
        aria-expanded={isOpen}
        aria-controls="menu"
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={handleKeyDown}
      >
        Menu
      </button>
      
      {isOpen && (
        <ul id="menu" role="menu">
          <li role="menuitem">Item 1</li>
          <li role="menuitem">Item 2</li>
        </ul>
      )}
    </div>
  );
}
```

---

### Tabs

**Expected keyboard behavior:**
- Tab: Move to tab list
- Arrow Left/Right: Switch tabs
- Tab again: Into panel content

```javascript
function Tabs() {
  const [activeTab, setActiveTab] = useState(0);
  
  const handleKeyDown = (e, index) => {
    switch (e.key) {
      case 'ArrowRight':
        setActiveTab((index + 1) % tabs.length);
        break;
      case 'ArrowLeft':
        setActiveTab((index - 1 + tabs.length) % tabs.length);
        break;
      case 'Home':
        setActiveTab(0);
        break;
      case 'End':
        setActiveTab(tabs.length - 1);
        break;
    }
  };
  
  return (
    <div role="tablist">
      {tabs.map((tab, i) => (
        <button
          key={i}
          role="tab"
          aria-selected={i === activeTab}
          tabIndex={i === activeTab ? 0 : -1}
          onKeyDown={(e) => handleKeyDown(e, i)}
        >
          {tab.label}
        </button>
      ))}
    </div>
  );
}
```

---

## 7. Testing Checklist

### Manual Testing

**Basic test:**
1. ☐ Disconnect mouse
2. ☐ Tab through entire page
3. ☐ Can reach all interactive elements?
4. ☐ Focus visible on all elements?
5. ☐ Tab order logical?

**Functional test:**
1. ☐ All buttons work with Enter/Space?
2. ☐ Forms submittable with keyboard?
3. ☐ Modals trap focus?
4. ☐ Modals close with Escape?
5. ☐ Menus navigable with arrows?

**Edge cases:**
1. ☐ Skip links work?
2. ☐ Custom shortcuts documented?
3. ☐ No keyboard traps?
4. ☐ Focus returns after modal close?

---

### Browser DevTools

```
Chrome DevTools:
1. Elements tab → Select element
2. Accessibility pane → "Computed Properties"
3. Check "Focusable" = true
4. Check "tabindex" value
```

---

## Tổng Kết

**Key Principles:**

1. **All Interactive = Focusable**
   - If you can click it, you can Tab to it

2. **Tab Order = DOM Order**
   - Avoid positive tabindex
   - Match visual to DOM order

3. **Focus Visible**
   - Never remove outline without replacement
   - Use `:focus-visible`

4. **Standard Shortcuts**
   - Enter/Space for buttons
   - Escape closes modals
   - Arrows for menus/tabs

5. **Test Without Mouse**
   - Unplug and try!
   - All features accessible?

---

**Next:** [Screen Readers →](./screen-readers.md)
