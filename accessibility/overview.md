# Accessibility Overview – Why & When

> **Mục tiêu:** Hiểu tại sao accessibility quan trọng, WCAG levels, và decision framework.

---

## 1. Why Accessibility Matters

### The Numbers

**Global statistics:**
- 15% dân số (1.3 billion người) có disabilities
- 253 million người visual impairment
- 466 million người hearing loss
- Temporary disabilities: Broken arm, bright sunlight, noisy environment

**Business impact:**
- Legal: ADA lawsuits tăng 300% (2018-2023)
- Market: Bỏ qua 15% potential users
- SEO: Accessible sites rank better
- UX: Accessibility = better UX cho tất cả

---

### Types of Disabilities

**1. Visual:**
- Blindness → Screen readers
- Low vision → Zoom, high contrast
- Color blindness → Don't rely on color alone

**2. Motor:**
- Cannot use mouse → Keyboard-only
- Limited mobility → Large click targets
- Tremors → No time-sensitive actions

**3. Auditory:**
- Deaf → Captions for videos
- Hard of hearing → Visual indicators

**4. Cognitive:**
- Dyslexia → Clear fonts, text spacing
- ADHD → Minimize distractions
- Memory issues → Clear navigation

**5. Temporary:**
- Broken arm → One-handed operation
- Bright sunlight → High contrast needed
- Loud environment → Visual indicators

---

## 2. WCAG Levels

### Web Content Accessibility Guidelines (WCAG 2.1)

**Three conformance levels:**

### Level A (Must Have) 🔴

**Minimum accessibility**
- If you fail Level A, some users CANNOT use your site

**Examples:**
- ✅ All images have alt text
- ✅ Form inputs have labels
- ✅ Color not the only way to convey info
- ✅ Keyboard accessible
- ✅ No seizure-inducing flashes

**Priority:** Critical for all websites

---

### Level AA (Should Have) 🟡

**Recommended for most websites**
- Legal requirement for many organizations (ADA, Section 508)
- Removes significant barriers

**Examples:**
- ✅ Color contrast ratio 4.5:1 (normal text)
- ✅ Text resizable to 200% without loss
- ✅ Focus visible
- ✅ Meaningful link text (not "click here")
- ✅ Multiple ways to navigate (menu, search, sitemap)

**Priority:** Target for most production apps

---

### Level AAA (Nice to Have) ⭐

**Enhanced accessibility**
- Very strict, not always feasible
- Good for specialized sites (government, healthcare)

**Examples:**
- ✅ Color contrast ratio 7:1
- ✅ Sign language videos
- ✅ Extended audio descriptions
- ✅ No time limits
- ✅ Context-sensitive help

**Priority:** Optional, unless required by industry

---

### Practical Target

```
Minimum viable: Level A
Production standard: Level AA
Specialized needs: Level AAA (selectively)
```

---

## 3. POUR Principles

### WCAG built on 4 principles:

**P = Perceivable**
```
Users must be able to perceive the information
- Alt text for images
- Captions for videos
- Sufficient color contrast
```

**O = Operable**
```
Users must be able to operate the interface
- Keyboard accessible
- Enough time to read/use content
- No seizure-inducing content
```

**U = Understandable**
```
Information and operation must be understandable
- Readable text
- Predictable behavior
- Input assistance (error messages)
```

**R = Robust**
```
Content must work with current/future technologies
- Valid HTML
- Compatible with assistive tech
- Works across browsers
```

---

## 4. Decision Framework

### When to prioritize a11y fixes?

**Priority 1: Blockers** 🔴
```
User CANNOT complete task
- Form without labels → screen reader users blocked
- Keyboard trap → keyboard users stuck
- No alt text on critical images

Fix: Immediately
```

**Priority 2: Significant barriers** 🟡
```
User CAN complete but with difficulty
- Low contrast text
- Missing focus indicators
- Poor heading structure

Fix: Next sprint
```

**Priority 3: Enhancements** 🟢
```
User can complete, but better experience possible
- ARIA live regions for notifications
- Skip links
- Better error messages

Fix: Backlog
```

---

### Quick Checklist (Before Launch)

**Level A essentials:**
- ☐ All images have alt text
- ☐ All form inputs have labels
- ☐ Site keyboard navigable
- ☐ No color-only information
- ☐ Video has captions (if applicable)

**Level AA recommended:**
- ☐ Color contrast passes (4.5:1)
- ☐ Focus visible on all interactive elements
- ☐ Heading hierarchy correct (h1 → h2 → h3)
- ☐ Link text meaningful
- ☐ Forms have error messages

**Testing:**
- ☐ Tab through entire page (keyboard)
- ☐ Run axe DevTools
- ☐ Test with screen reader (basic)
- ☐ Lighthouse a11y score >90

---

## 5. Common Myths

### ❌ Myth 1: "Our users don't have disabilities"

**Reality:**
- 15% of population has disabilities
- Temporary disabilities affect everyone
- Accessible design benefits all users

---

### ❌ Myth 2: "Accessibility is expensive"

**Reality:**
- Cheaper when built in from start
- Lawsuit costs > development costs
- Many fixes are simple (add alt text, labels)

**Cost comparison:**
```
Build accessible from start: +10-15% dev time
Retrofit later: +50-100% dev time
Lawsuit settlement: $50k-$500k+
```

---

### ❌ Myth 3: "Automated tools catch everything"

**Reality:**
- Automated tools: ~30-40% of issues
- Manual testing required for:
  - Keyboard navigation flow
  - Screen reader experience
  - Cognitive load
  - Content quality

---

### ❌ Myth 4: "It's ugly"

**Reality:**
- Accessible ≠ ugly
- Good design is accessible design
- Examples: GitHub, Stripe, GOV.UK

---

## 6. ROI of Accessibility

### Benefits Beyond Compliance

**1. SEO boost**
```
Semantic HTML → Better crawling
Alt text → Image search
Clear structure → Better rankings
```

**2. Better mobile experience**
```
Large tap targets → Easier on mobile
Keyboard accessible → Works with external keyboards
Clear labels → Voice control works better
```

**3. Performance**
```
Semantic HTML → Less JavaScript
Simpler markup → Faster load
Progressive enhancement → Works on slow connections
```

**4. Maintainability**
```
Clear structure → Easier to update
Standard patterns → Easier onboarding
Less hacks → Fewer bugs
```

**5. Brand reputation**
```
Inclusive brand image
Positive press
User loyalty
```

---

## 7. Getting Started

### Phase 1: Foundation (Week 1-2)

```
1. Learn basics
   - Read WCAG 2.1 quick reference
   - Watch "How People with Disabilities Use the Web"

2. Audit current site
   - Run axe DevTools
   - Test keyboard navigation
   - Try basic screen reader

3. Fix critical issues (Level A)
   - Add alt text
   - Add form labels
   - Fix keyboard traps
```

---

### Phase 2: Standard Compliance (Month 1-2)

```
1. Achieve Level AA
   - Fix color contrast
   - Improve focus indicators
   - Fix heading hierarchy

2. Team training
   - Accessibility basics
   - How to test
   - Common patterns

3. Process integration
   - Add a11y to code review
   - Add to Definition of Done
   - Automated testing in CI
```

---

### Phase 3: Continuous Improvement

```
1. Regular audits
   - Quarterly accessibility review
   - User testing with disabled users

2. Stay updated
   - WCAG 2.2, 3.0 changes
   - New assistive technologies

3. Advocate
   - Share knowledge
   - Accessibility champions
   - Company-wide awareness
```

---

## 8. Resources

### Official Standards
- WCAG 2.1: https://www.w3.org/WAI/WCAG21/quickref/
- ARIA Authoring Practices: https://www.w3.org/WAI/ARIA/apg/

### Testing Tools
- axe DevTools (browser extension)
- Lighthouse (built into Chrome)
- WAVE (online checker)

### Learning
- WebAIM articles
- A11y Project
- Deque University (courses)

### Communities
- a11y Slack
- Twitter: #a11y
- Stack Overflow [accessibility]

---

## Tổng Kết

**Key Takeaways:**

1. **Accessibility = Civil Right**
   - Not optional, not nice-to-have
   - Legal + ethical responsibility

2. **Target Level AA**
   - Level A = minimum
   - Level AA = standard
   - Level AAA = selective

3. **Build In, Don't Bolt On**
   - Cheaper in design phase
   - Better UX for everyone
   - Easier to maintain

4. **Test, Test, Test**
   - Automated tools (30%)
   - Manual testing (70%)
   - Real users (invaluable)

5. **It's a Journey**
   - Start with critical issues
   - Improve incrementally
   - Never "done"

---

**Next Steps:**
- Read: [Semantic HTML →](./semantic-html.md)
- Practice: Tab through your site
- Install: axe DevTools
- Test: Basic screen reader

🚀 **Remember:** Good accessibility = good UX for everyone.
