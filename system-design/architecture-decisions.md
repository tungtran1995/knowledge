# Architecture Decisions - Concept Guide

> **Core idea:** Document WHY you chose X over Y, so future you (or teammates) don't re-debate the same decision.

---

## 1. What is an Architecture Decision?

### Definition

```
Architecture Decision = A choice between multiple valid options
                        that impacts system structure
                        and is hard to reverse
```

**Examples:**
- Redux vs Zustand vs MobX
- Horizontal vs Vertical code structure
- REST vs GraphQL
- Client-side routing vs SSR

**NOT architecture decisions:**
- Variable naming
- Comment style
- One-off bug fixes

---

## 2. Why Document Decisions?

### The Problem

```
Month 1: Team debates Redux vs Zustand → Choose Zustand
Month 6: New dev joins → "Why Zustand? Can we use Redux?"
         Team re-debates same decision → Wastes time
```

---

### The Solution

```
Month 1: Decide Zustand → Write ADR (Architecture Decision Record)
Month 6: New dev asks → "Read ADR-003"
         No re-debate → Decision explained with context
```

**Benefit:** **Preserve context** → Prevent rehashing → Save time.

---

## 3. ADR Template (Simple)

```markdown
# ADR-XXX: [Decision Title]

## Context
What problem are we solving?
What constraints do we have?

## Decision
What did we choose?

## Consequences
What do we gain?
What do we lose?

## Alternatives
What else did we consider?
Why rejected?
```

---

## 4. Key Architecture Decision Types

### Type 1: State Management

**Question:** How do we manage global state?

**Options:**
- useState + Context (built-in, simple)
- Zustand (minimal boilerplate)
- Redux (mature, tooling)
- Jotai (atomic state)

**Decision factors:**
- Team size (small team → simpler tool)
- State complexity (simple → Context ok, complex → Redux)
- Learning curve (junior team → avoid MobX magic)

---

**Example Decision:**

```
Context: 5 devs, medium complexity, fast iteration needed
Decision: Zustand
Reason: Balance simplicity + scalability, less boilerplate than Redux
Trade-off: Smaller ecosystem (acceptable for our scale)
```

---

### Type 2: Code Organization

**Question:** How do we organize files?

**Options:**

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
  dashboard/
    components/
    hooks/
```

**Decision factors:**
- Codebase size (<10K → horizontal ok, >50K → vertical better)
- Team structure (feature teams → vertical)

---

**Example Decision:**

```
Context: 40K LOC, 10 devs, 3 feature teams, merge conflicts frequent
Decision: Vertical (feature-based)
Reason: Co-location, clear ownership, fewer merge conflicts
Trade-off: Migration effort (2 weeks) < long-term maintainability
```

---

### Type 3: Data Fetching

**Question:** How do we fetch & cache data?

**Options:**
- Manual fetch + useState
- React Query / SWR (smart caching)
- Redux Toolkit Query (integrated with Redux)

**Decision factors:**
- Cache complexity (simple → manual ok, complex → React Query)
- Real-time needs (yes → React Query + WebSocket)
- Existing state lib (have Redux → RTK Query makes sense)

---

**Example Decision:**

```
Context: News app, data stale for 5min ok, mobile users (flaky network)
Decision: React Query with stale-while-revalidate
Reason: Instant UI (return cache) + auto background refresh
Trade-off: Data can be 5min stale (acceptable for news)
```

---

### Type 4: Routing Strategy

**Question:** Client-side routing vs Server-side rendering?

**Options:**
- Client-side (React Router)
- SSR (Next.js, Remix)
- SSG (Static Site Generation)

**Decision factors:**
- SEO important? (yes → SSR/SSG)
- Initial load time? (critical → SSR/SSG)
- Dynamic data? (high → SSR, low → SSG)

---

**Example Decision:**

```
Context: E-commerce site, SEO critical, dynamic product data
Decision: Next.js with ISR (Incremental Static Regeneration)
Reason: SEO + fast static pages + revalidate stale products
Trade-off: Server costs (acceptable, revenue dependent on SEO)
```

---

### Type 5: Real-time Communication

**Question:** How do we handle real-time updates?

**Options:**
- Polling (simple, high latency)
- Server-Sent Events / SSE (one-way, efficient)
- WebSocket (two-way, complex)

**Decision factors:**
- Update frequency (low → polling ok, high → WebSocket)
- Bidirectional? (no → SSE, yes → WebSocket)
- Scale (1K users → any, 100K → WebSocket + infrastructure)

---

**Example Decision:**

```
Context: Chat app, bidirectional, 10K concurrent users
Decision: WebSocket with Socket.io
Reason: Bidirectional real-time required, Socket.io handles reconnect
Trade-off: Infrastructure complexity (need WebSocket server scaling)
```

---

## 5. Decision-Making Framework

### Step 1: Define Context

```
Ask:
- What problem are we solving?
- What are our constraints? (team, time, budget, scale)
- What are our requirements? (must-have vs nice-to-have)
```

---

### Step 2: List Options

```
Ask:
- What are valid alternatives?
- What's the default/obvious choice?
- What's the innovative/modern choice?
```

**Tip:** Usually 3-5 options. Too many = analysis paralysis.

---

### Step 3: Evaluate Trade-offs

```
For each option:
- Pros (what we gain)
- Cons (what we lose)
- Fit with context (team skill, scale, timeline)
```

**Example:**

| Option | Pros | Cons | Fit |
|--------|------|------|-----|
| Redux | Mature, tooling | Verbose | Overkill for 5 devs |
| Zustand | Simple, fast | Small ecosystem | ✅ Good fit |
| MobX | Minimal code | Magic, hard debug | Team prefers explicit |

---

### Step 4: Decide

```
Pick option that:
✅ Solves problem
✅ Fits constraints
✅ Team can execute
```

**Not necessarily:**
- ❌ "Best" option (context-dependent)
- ❌ Most popular (hype ≠ fit)
- ❌ Most innovative (boring tech ok)

---

### Step 5: Document

```
Write ADR:
- Context (why we decided)
- Decision (what we chose)
- Consequences (trade-offs)
- Alternatives (what we rejected)
```

**Store:** Git repo (`docs/adr/`), wiki, Notion.

---

## 6. Common Pitfalls

### Pitfall 1: Deciding without context

```
❌ "Let's use Redux because Google uses it"
✅ "Google has 1000 devs, we have 5. Zustand fits better."
```

**Fix:** Always document **your context**, not generic best practices.

---

### Pitfall 2: Not revisiting decisions

```
❌ Decision made 2 years ago, context changed (team grew 5→50)
   Still using Zustand → now bottleneck
```

**Fix:** Add **review date** to ADRs (e.g., every 6-12 months).

---

### Pitfall 3: Over-engineering

```
❌ "Let's use micro-frontends" (5 devs, 20K LOC app)
✅ "Monolith sufficient until 50K+ LOC or multiple teams"
```

**Fix:** Choose **simplest solution** that solves problem. Complexity has cost.

---

### Pitfall 4: Bikeshedding

```
❌ Spending 2 hours debating Redux vs Zustand (both work)
✅ Spend 30min, pick one, document, move on
```

**Fix:** Set **time limit** for decision. Perfect is enemy of good.

---

## 7. Mental Models

### Model 1: Decision Irreversibility

```
Easy to reverse:     Don't overthink (e.g., CSS framework)
Hard to reverse:     Document carefully (e.g., state management)
Very hard to reverse: Get consensus (e.g., monolith vs micro-frontends)
```

**Rule:** Effort spent deciding ∝ Cost of reversal.

---

### Model 2: Decision Impact Radius

```
Local impact:   One component → Junior can decide
Team impact:    One feature → Team lead decides
System impact:  Whole codebase → Architect decides
```

**Rule:** Bigger impact = Higher level approver.

---

### Model 3: Context Decay

```
Time = 0:  Everyone knows why we chose X
Time = 6m: Half remember
Time = 2y: Nobody remembers → Re-debate starts

→ Write it down NOW
```

**Rule:** Document while context is fresh.

---

## 8. Practical Examples

### Example 1: State Management

```markdown
# ADR-001: Zustand for Global State

Context: 5 devs, 30K LOC, medium state complexity
Decision: Zustand
Why: Less boilerplate than Redux, sufficient for our scale
Trade-off: Smaller ecosystem (acceptable)
Review: 2024-07-01 (if team grows to 15+, reconsider Redux)
```

---

### Example 2: Real-time Updates

```markdown
# ADR-002: Polling for Notifications

Context: Low-traffic app (100 users), simple notifications
Decision: Poll /api/notifications every 30s
Why: Simplest solution, no WebSocket infrastructure needed
Trade-off: 30s latency (acceptable, not chat)
Upgrade trigger: If traffic > 1K users or latency unacceptable
```

---

### Example 3: Code Organization

```markdown
# ADR-003: Keep Horizontal Structure (For Now)

Context: 8K LOC, 3 devs, no merge conflicts yet
Decision: Keep components/, hooks/, utils/
Why: Simple, team familiar, no pain points yet
Trade-off: May need vertical later (monitor: if LOC > 30K)
Review: 2024-06-01 or when team grows to 8+
```

---

## 9. Quick Reference

### When to Write ADR?

```
✅ Choosing state management library
✅ Changing code structure (horizontal → vertical)
✅ Adopting new major dependency
✅ Architectural pattern change (REST → GraphQL)
✅ Data fetching strategy

❌ One-off bug fix
❌ Styling tweak
❌ Variable renaming
```

---

### ADR Checklist

```
☐ Context clearly stated (problem + constraints)
☐ Decision explicit (what we chose)
☐ Consequences listed (pros + cons)
☐ Alternatives documented (what we rejected + why)
☐ Review date set (when to reconsider)
☐ Approved by appropriate level (team lead / architect)
```

---

## 10. Tổng Kết

### Core Principles

1. **Context is king:** Same decision, different context = different choice
2. **Document trade-offs:** No perfect solution, only acceptable trade-offs
3. **Revisit decisions:** Context changes → Decision may need revision
4. **Simplicity first:** Boring tech that works > exciting tech that's overkill

---

### Mental Model

```
Architecture Decision = 
  Context (problem + constraints)
  + Options (alternatives)
  + Trade-offs (pros vs cons)
  + Decision (chosen option)
  + Rationale (why this fits our context)
```

---

### Key Insight

> **Good architecture = Making reversible decisions reversible, and irreversible decisions carefully.**

**Reversible:** CSS framework, UI library (easy to change)  
**Irreversible:** State management, routing strategy, code structure (hard to change)

→ **Spend time proportional to reversal cost.**

---

**Bottom Line:**

> Architecture decisions aren't about finding "the best" solution.  
> They're about finding **the right solution for YOUR context**,  
> and **documenting WHY** so future you doesn't waste time re-debating.

**Action:** Next time you debate Redux vs Zustand → Write ADR → Save 10 hours in 6 months.