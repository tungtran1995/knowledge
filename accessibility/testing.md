# Accessibility Testing – Tools & Methods

> **Reality check:** Automated tools catch ~30-40% of issues. Manual testing required.

---

## 1. Testing Strategy

### The Testing Pyramid

```
          Manual Testing (10%)
         Real user testing with disabilities
      
        Manual Expert Review (20%)
       Screen reader, keyboard, cognitive

      Automated Testing (70%)
     axe, Lighthouse, pa11y in CI
```

**All three layers needed for comprehensive coverage.**

---

## 2. Automated Tools

### axe DevTools (Best Free Tool)

**Install:** Browser extension (Chrome, Firefox, Edge)

**How to use:**
1. Open DevTools → "axe DevTools" tab
2. Click "Scan ALL of my page"
3. Review violations

**What it catches:**
- ✅ Missing alt text
- ✅ Missing form labels
- ✅ Color contrast issues
- ✅ ARIA errors
- ✅ Heading hierarchy
- ✅ Keyboard traps (some)

**What it DOESN'T catch:**
- ❌ Alt text quality (is it descriptive?)
- ❌ Keyboard navigation flow
- ❌ Screen reader UX
- ❌ Cognitive load issues

---

### Lighthouse (Built into Chrome)

**How to use:**
1. Open DevTools → "Lighthouse" tab
2. Select "Accessibility"
3. Click "Generate report"

**Provides:**
- Overall score (0-100)
- Categorized issues
- Links to learn more

**Good for:**
- Quick health check
- CI/CD integration
- Before/after comparisons

**Limitation:** Subset of axe checks.

---

### WAVE (WebAIM)

**Use:** https://wave.webaim.org or browser extension

**Visual feedback:**
- Icons overlaid on page
- Shows where errors are
- Color coding (errors, alerts, features)

**Good for:**
- Visual learners
- Quick scan
- Explaining to non-technical team

---

### pa11y (CLI Tool)

```bash
npm install -g pa11y

# Test single page
pa11y https://example.com

# Test with different standard
pa11y --standard WCAG2AA https://example.com

# JSON output for CI
pa11y --reporter json https://example.com > report.json
```

**Good for:**
- CI/CD pipelines
- Automated regression testing
- Bulk scanning multiple pages

---

### Accessibility Insights (Microsoft)

**Features:**
- Automated checks
- Guided manual testing
- Tab order visualization
- Color contrast analyzer

**Unique:** Guided assessment (walks through WCAG checks).

---

## 3. Manual Keyboard Testing

### Basic Test (5 minutes)

**Checklist:**

1. ☐ **Disconnect mouse** (force keyboard-only)

2. ☐ **Tab through page**
   - Can reach all interactive elements?
   - Tab order logical?

3. ☐ **Focus visible**
   - Can you see where focus is?
   - Focus indicator clear on all elements?

4. ☐ **Activate elements**
   - Buttons work with Enter/Space?
   - Links work with Enter?

5. ☐ ** Test modals**
   - Opens with keyboard?
   - Focus trapped inside?
   - Closes with Escape?
   - Focus returns to trigger?

6. ☐ **Test forms**
   - Tab through all fields?
   - Can submit with keyboard?
   - Error validation accessible?

---

### Advanced Keyboard Testing

**Dropdowns:**
- Arrow keys navigate options?
- Enter/Space select?
- Escape closes?

**Tabs:**
- Arrow keys switch tabs?
- Tab moves to panel content?

**Carousels:**
- Arrow keys navigate slides?
- Can pause auto-play?

**Data tables:**
- Tab through interactive cells?
- Arrow keys navigate (if applicable)?

---

## 4. Screen Reader Testing

### Quick Test (15 minutes)

**Setup:**
- Mac: VoiceOver (Cmd+F5)
- Windows: NVDA (free download)

**Test flow:**

1. ☐ **Navigate by headings** (H key or VO+Cmd+H)
   - Headings in logical order?
   - Make sense out of context?

2. ☐ **Navigate by links** (K key or rotor)
   - Link text descriptive?
   - No "click here" links?

3. ☐ **Navigate by landmarks** (D key or rotor)
   - Main landmark exists?
   - Navigation clearly labeled?

4. ☐ **Forms**
   - All inputs have labels?
   - Labels read before input?
   - Required fields announced?
   - Errors announced?

5. ☐ **Images**
   - Alt text descriptive?
   - Decorative images skipped (empty alt)?

6. ☐ **Buttons**
   - Purpose clear from announcement?
   - Icon buttons labeled?

---

### Common Screen Reader Commands

**VoiceOver (Mac):**
```
Cmd+F5: Start/stop
VO+A: Start reading
VO+→: Next item
VO+Cmd+H: Next heading
VO+U: Rotor (navigation menu)
Ctrl: Stop reading
```

**NVDA (Windows):**
```
Ctrl+Alt+N: Start
Insert+↓: Read all
H: Next heading
K: Next link
B: Next button
Insert+F7: Elements list
Ctrl: Stop reading
```

---

## 5. Color Contrast Testing

### Tools

**Browser extensions:**
- axe DevTools (automatic scan)
- Accessibility Insights (point and click)

**Online:**
- WebAIM Contrast Checker
- Contrast Ratio by Lea Verou

**Design tools:**
- Figma: Stark plugin
- Sketch: Contrast plugin

---

### WCAG Requirements

**Level AA (standard):**
- Normal text: 4.5:1
- Large text (18pt+ or 14pt+ bold): 3:1

**Level AAA (enhanced):**
- Normal text: 7:1
- Large text: 4.5:1

**Testing:**
```
1. Identify all text on page
2. Check each text/background combination
3. Flags any below 4.5:1 (normal) or 3:1 (large)
4. Fix with darker text or lighter background
```

---

### Common Failures

```
❌ Light gray text on white: 2.5:1 (fails)
✅ Dark gray text on white: 4.6:1 (passes)

❌ Yellow on white: 1.2:1 (fails badly)
✅ Dark yellow/gold on white: 5.1:1 (passes)

❌ Blue link on dark blue: 2.1:1 (fails)
✅ Blue link on white: 8:1 (passes)
```

---

## 6. Testing Checklist (Pre-Launch)

### Automated (10 minutes)

- ☐ Run axe DevTools → 0 violations
- ☐ Lighthouse a11y score ≥90
- ☐ pa11y in CI passing

---

### Manual Keyboard (15 minutes)

- ☐ Unplug mouse, tab through site
- ☐ All interactive elements reachable
- ☐ Tab order logical
- ☐ Focus always visible
- ☐ All actions work with keyboard

---

### Screen Reader (15 minutes)

- ☐ Headings complete and ordered
- ☐ All form inputs labeled
- ☐ All images have alt text
- ☐ Link text descriptive
- ☐ Buttons have accessible names

---

### Visual (10 minutes)

- ☐ Color contrast passing (4.5:1)
- ☐ UI works at 200% zoom
- ☐ No information conveyed by color alone
- ☐ Text spacing comfortable

---

## 7. CI/CD Integration

### GitHub Actions Example

```yaml
name: Accessibility Tests

on: [push, pull_request]

jobs:
  a11y:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm install
      
      - name: Build
        run: npm run build
      
      - name: Start server
        run: npm run serve &
        
      - name: Wait for server
        run: npx wait-on http://localhost:3000
      
      - name: Run pa11y
        run: |
          npm install -g pa11y
          pa11y http://localhost:3000
```

---

### Fail Build on Errors

```javascript
// pa11y config
module.exports = {
  standard: 'WCAG2AA',
  threshold: 0, // Fail on any errors
  ignore: [
    // Ignore specific rules if needed (with justification!)
  ]
};
```

---

## 8. User Testing with Real Users

### Why It Matters

**Automated + manual testing ≠ real experience**

Real users reveal:
- Confusing workflows
- Missing context
- Cognitive load issues
- Unexpected use cases

---

### How to Recruit

**Resources:**
- AccessibilityOz
- Fable (remote user testing)
- Local disability organizations
- University accessibility programs

**Pay fairly:** $50-100/hour typical.

---

### Testing Protocol

1. **Task-based:** "Book a flight to NYC"
2. **Think-aloud:** User narrates thoughts
3. **Observer notes:** Issues encountered
4. **Follow-up questions:** Why confusing?

**Don't:**
- Lead participants
- Interrupt problem-solving
- Explain how to use it

---

## 9. Regression Testing

### Prevent Accessibility Regressions

**Strategy:**
```
1. Automated tests in CI (pa11y, axe-core)
2. Manual spot checks on major features
3. Quarterly full audits
4. User testing before major releases
```

---

### axe-core in Jest

```javascript
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

test('Button is accessible', async () => {
  const { container } = render(<Button>Click me</Button>);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

---

## 10. Common Testing Mistakes

### ❌ Mistake 1: Only automated testing

**Reality:** Catches 30-40%, misses most UX issues.

---

### ❌ Mistake 2: Testing with mouse

**Why wrong:** Keyboard/SR users can't use mouse.

---

### ❌ Mistake 3: One-time audit

**Reality:** Accessibility = continuous (like security).

---

### ❌ Mistake 4: Fixing score, not experience

**Issue:** Can have 100/100 and still be unusable.

---

## Tổng Kết

**Testing Strategy:**

```
Daily: Automated tests in CI
Weekly: Manual keyboard spot checks
Monthly: Screen reader testing on new features
Quarterly: Full expert audit
Annually: User testing with disabled users
```

**Tools Priority:**

1. **Must have:** axe DevTools (free)
2. **Should have:** Lighthouse (built-in)
3. **Nice to have:** pa11y (CI), WAVE, Accessibility Insights

**Testing Budget:**

```
Minimum (small team):
- Automated: Free (axe, Lighthouse)
- Manual: 1-2 hours/sprint
- Total: ~5% dev time

Recommended (production app):
- Automated: Free
- Manual: 2-3 hours/sprint
- Quarterly audit: $2-5k
- User testing: $1-2k/year
- Total: ~10% dev time
```

**Remember:** Perfect is the enemy of good. Start with automated tests, add manual testing incrementally.

---

**All accessibility docs complete!** 🎉

**Quick Links:**
- [Overview](./overview.md)
- [Semantic HTML](./semantic-html.md)
- [ARIA Guide](./aria-guide.md)
- [Keyboard Nav](./keyboard-nav.md)  
- [Screen Readers](./screen-readers.md)
- [Testing](./testing.md) (you are here)
