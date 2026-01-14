# React Core - Deep Dive

> Tài liệu chuyên sâu về React Core Concepts từ cơ bản đến nâng cao

---

## Mục Lục

### I. VIRTUAL DOM & RECONCILIATION
1. Virtual DOM và Reconciliation
2. Fiber Architecture
3. Diffing Algorithm
4. Key Prop

### II. RENDERING LIFECYCLE
1. Render Phase vs Commit Phase
2. Lifecycle Tổng Quát
3. Effects (useEffect / useLayoutEffect)
4. useLayoutEffect vs useEffect
5. State Updates & Batching
6. Concurrent Rendering

### III. HOOKS DEEP DIVE
1. useState
2. useEffect
3. useCallback / useMemo
4. useRef
5. Custom Hooks

### IV. PERFORMANCE OPTIMIZATION
1. React.memo
2. useMemo vs useCallback
3. Code Splitting
4. Profiler API
5. DevTools Profiler
6. Mental Model Tổng Hợp

### V. CONTEXT API
1. When to Use Context
2. Performance Implications
3. Provider Composition
4. Avoiding Re-renders
5. Advanced Patterns
6. Performance Debugging Checklist
7. Real-World Example: Form Context
8. Mental Model Tổng Hợp
9. Key Principles - Principal Level

---

# I. VIRTUAL DOM & RECONCILIATION

---

### Overall strategy (thứ tự ưu tiên)

- **Virtual DOM là representation**, không phải magic performance boost
- Fiber = **interruptible reconciliation**, không phải multi-threading
- **Render ≠ Commit** ≠ DOM update
- Hiểu đúng **2-tree architecture** (current & WIP)

---

### Key decision rules

- Render phase:
  - Phải pure, idempotent
  - Có thể bị discard/abort
  - Không side effects
- Commit phase:
  - Synchronous, atomic
  - Apply DOM mutations
  - Run effects
- Identity (key + type) quyết định:
  - Reuse vs remount
  - State preservation

---

### Common misconceptions

- "Virtual DOM nhanh hơn Real DOM" → sai framing
- Fiber = multi-thread → sai, vẫn single thread
- React Element = Fiber → sai, Element là output, Fiber là execution unit
- Render = DOM update → sai, render chỉ là tính toán
- Stack reconciler có thể patch → không, do bản chất call stack

---

## 1. Virtual DOM — Representation, Not Performance

### Deep explanation

#### A. Bản chất

- Virtual DOM không phải là DOM và không phải bản sao của DOM. Virtual DOM là **immutable JavaScript object** mô tả "UI nên trông như thế nào" tại một thời điểm cụ thể.
- Mục đích chính:
  - Tách biệt **representation** (React Element tree) khỏi **implementation** (Real DOM)
  - Cho phép React tính toán trong render phase mà không đụng tới DOM
  - Batch và optimize DOM mutations trong commit phase

#### B. Cơ chế

- Mỗi lần render, React tạo một **React Element tree hoàn toàn mới** (Virtual DOM tree mới).
- React so sánh (diff) tree mới với tree cũ để tính ra **minimal set of DOM mutations** cần thiết.
- DOM chỉ được cập nhật **một lần duy nhất trong commit phase**, với tất cả mutations được batch lại.
- Flow hoàn chỉnh:
  - State change
  - → Tạo new element tree
  - → Diff với old tree
  - → Compute mutations
  - → Commit to DOM (batched)

#### C. Hệ quả / Pitfalls

- **Sai lầm phổ biến**: "Virtual DOM nhanh hơn Real DOM"
  - Đây là **sai framing**. Virtual DOM không tự nó nhanh hơn.
  - Cái nhanh thực sự là:
    - JS object diff (rẻ hơn DOM operations)
    - Batching mutations (giảm reflow/repaint)
    - Tránh DOM mutations không cần thiết
- Nếu hiểu sai → nghĩ Virtual DOM là "silver bullet" cho performance
- Thực tế: Virtual DOM là **trade-off** - đổi memory + CPU cho JS diff để có declarative API và predictable updates

#### D. Khi nào dùng / Khi nào tránh

- Virtual DOM không phải lựa chọn – nó là core của React
- Nhưng hiểu đúng giúp:
  - Biết render phase chỉ tạo elements + tính toán, **không đụng DOM**
  - Mọi logic cần đụng DOM phải ở **commit phase hoặc effects**
  - Verify bằng DevTools Performance:
    - JS time = render phase
    - Layout/Paint = commit phase
  - Nếu DOM mutate quá nhiều → reconciliation chưa hiệu quả (kiểm tra key, component structure)

---

### TL;DR / Bullet summary

- Virtual DOM = representation (JS object), không phải performance magic
- Performance đến từ: diff + batching + commit separation
- Render phase = tạo elements, commit phase = update DOM
- Hiểu đúng để tránh premature optimization

---

## 2. React Element vs Fiber

### Deep explanation

#### A. Bản chất

- **React Element** là output của JSX/`React.createElement`, mô tả "UI trông như thế nào" (immutable object).
- **Fiber** là **unit of work** – cấu trúc dữ liệu nội bộ React dùng để schedule và reconcile.
- Element là **what** (what to render), Fiber là **how** (how to execute render).

#### B. Cơ chế

- JSX compile thành `React.createElement(type, props, children)`
- `React.createElement` trả về **React Element** (plain object)
- React nhận Element và tạo/cập nhật **Fiber node** tương ứng
- Fiber giữ:
  - State (memoizedState)
  - Hooks linked list
  - Priority (lanes)
  - Effect flags (Placement, Update, Deletion)
- Flow: `JSX → React Element → Fiber`

#### C. Hệ quả / Pitfalls

- **Nhầm lẫn phổ biến**: nghĩ React Element là Fiber
- Hệ quả:
  - Hiểu sai bailout (bailout xảy ra ở Fiber level, không phải Element level)
  - Hiểu sai reuse (reuse Fiber node, không reuse Element)
  - Hiểu sai render vs commit (render tạo Element, commit apply Fiber effects)
- Element được tạo mới **mỗi render**, nhưng Fiber có thể **reuse** (nếu key + type khớp)

#### D. Khi nào dùng / Khi nào tránh

- Không phải "khi nào dùng" – đây là concept để hiểu React internals
- Giá trị thực tế:
  - Khi debug: Element là output của component function, Fiber là runtime state
  - Khi optimize: Focus vào Fiber reuse (key, type stability), không focus vào Element creation cost (rất rẻ)

---

### TL;DR / Bullet summary

- Element = immutable output (what to render)
- Fiber = mutable work unit (how to execute)
- Element tạo mới mỗi render, Fiber có thể reuse
- Hiểu đúng để debug và optimize đúng chỗ

---

## 3. Stack Reconciler — Why It Failed

### Deep explanation

#### A. Bản chất

- React ≤15 sử dụng **stack reconciler** – reconciliation hoàn toàn dựa vào JS call stack.
- Đặc điểm:
  - Synchronous (đồng bộ)
  - Recursive (đệ quy)
  - Non-interruptible (không thể pause/resume)
  - Không có khái niệm priority

#### B. Cơ chế

- Render theo DFS (depth-first search) traversal:
  - `render(App) → render(A) → render(B) → render(C) → ...`
- Toàn bộ logic nằm trong call stack của JS
- Một khi bắt đầu render → **phải chạy hết**, không thể yield giữa chừng
- Main thread bị chiếm trọn cho đến khi render xong

#### C. Hệ quả / Pitfalls

- **UI freeze**: Component tree lớn → render lâu → main thread bị block → UI đơ
- **Không priority**: Mọi update được xử lý như nhau. Typing (urgent) và analytics (non-urgent) cùng priority → user input bị lag
- **Input/animation bị block**: Trong lúc render, browser không xử lý được events hay paint frames
- **Stack overflow**: Tree quá sâu → recursion quá sâu → tràn stack
- **Debug khó**: Call stack lớn, khó trace

#### D. Tại sao không thể patch

- Vấn đề nằm ở **model**, không phải implementation:
  - JS call stack không thể serialize (biến local nằm trong stack frames)
  - Không có điểm an toàn để pause/resume (mid-function pause → lose context)
  - Không cách nào "save progress" và "resume later"
- Cần kiến trúc hoàn toàn mới → Fiber

---

### TL;DR / Bullet summary

- Stack reconciler = synchronous, recursive, non-interruptible
- Hệ quả: UI freeze, no priority, input lag
- Không thể patch vì JS call stack không serialize được
- Cần model mới → Fiber

---

## 4. Fiber Architecture — Interruptible Reconciliation

### Deep explanation

#### A. Bản chất

- Fiber cho phép reconciliation trở thành **asynchronous** và **interruptible**.
- Fiber giải quyết vấn đề của stack reconciler bằng cách:
  - Chia work thành **units of work** (mỗi Fiber = 1 unit)
  - Cho phép **pause, resume, abort** work
  - Ưu tiên work dựa trên priority

#### B. Cơ chế

- Thay vì dựa vào call stack, Fiber tạo **linked list structure** của work units:
  - Mỗi Fiber node giữ reference tới: child, sibling, return (parent)
  - Scheduler quyết định: làm tiếp hay yield cho high-priority work
- Flow:
  - Bắt đầu unit of work
  - Scheduler check: có high-priority work không?
  - Nếu có → pause, xử lý high-priority, resume sau
  - Nếu không → tiếp tục
- Ví dụ: `Unit 1 → Pause → Handle event → Resume → Unit 2 → ...`

#### C. Hệ quả / Pitfalls

- **Sai lầm phổ biến**: nghĩ Fiber = multi-threading
  - **Sai**. Vẫn là **single JS thread**.
  - Fiber chỉ là **execution model** cho phép yield control, không phải parallel execution
- Hệ quả code:
  - Code trong render phase **phải pure, restartable, abortable**
  - Render có thể bị call nhiều lần hoặc discard → không được side effects
  - Commit phase vẫn synchronous (không thể interrupt DOM updates)

#### D. Khi nào có giá trị

- Fiber không phải API để "dùng" – nó là nền tảng cho:
  - Concurrent rendering
  - Priority-based updates
  - Time slicing
  - Suspense, Transitions
- Giá trị cho developer:
  - UI luôn responsive (không freeze)
  - Urgent updates (typing) không bị chặn bởi heavy renders
  - Cho phép build UX tốt hơn với concurrent features

---

### TL;DR / Bullet summary

- Fiber = interruptible reconciliation (vẫn single thread)
- Chia work thành units, có thể pause/resume/abort
- Render phase phải pure vì có thể bị discard
- Enable concurrent features, priority scheduling

---

## 5. Current Tree & Work-In-Progress Tree

### Deep explanation

#### A. Bản chất

- React Fiber luôn duy trì **2 Fiber trees** song song:
  - **Current tree**: tree đang hiển thị trên UI
  - **Work-in-progress (WIP) tree**: tree đang được xây dựng trong render phase
- Mỗi Fiber node có pointer `alternate` liên kết giữa current và WIP version của chính nó.

#### B. Cơ chế

- Khi update xảy ra:
  - React tạo/reuse WIP fibers dựa trên current fibers
  - Render từng WIP fiber (diff props/state, gắn effect flags)
  - Sau khi render phase hoàn tất và commit phase apply xong
  - **WIP tree trở thành current tree** (swap pointers)
- `fiber.alternate`:
  - `currentFiber.alternate === wipFiber`
  - `wipFiber.alternate === currentFiber`
- Cho phép so sánh props/state cũ (current) vs mới (WIP) mà không cần clone toàn bộ tree

#### C. Hệ quả / Pitfalls

- **Tại sao 2 trees?**
  - Không mutate current tree → UI hiện tại luôn ổn định
  - Cho phép rollback nếu render bị abort (discard WIP, current không đổi)
  - Tránh clone toàn bộ tree (expensive) – chỉ cần reuse và update WIP fibers
- **Pitfall**:
  - Mutate state/ref trong render phase → làm hỏng concurrent guarantees
  - Vì render có thể bị discard, mutations sẽ "leak" trạng thái không nhất quán

#### D. Khi nào quan trọng

- Khi hiểu concurrent rendering:
  - WIP là "render attempt" – có thể bị discard
  - Current là "committed UI" – luôn consistent
- Khi debug:
  - Nếu thấy render chạy nhiều lần nhưng UI không update → WIP bị discard, current không thay đổi
  - Nếu state "bị mất" → kiểm tra có mutate trong render không

---

### TL;DR / Bullet summary

- 2 trees: current (UI hiện tại) & WIP (render attempt)
- `fiber.alternate` liên kết giữa 2 versions
- Tránh clone tree, cho phép rollback, giữ UI stable
- Không mutate trong render → bảo vệ concurrent guarantees

---

## 6. Render Phase vs Commit Phase

### Deep explanation

#### A. Bản chất

- React update luôn có **2 phase tách biệt**:
  - **Render phase**: tính toán (interruptible)
  - **Commit phase**: apply changes (synchronous, atomic)
- **Render ≠ Commit**. Không phải mọi render đều dẫn tới commit.

#### B. Cơ chế

**Render Phase (Interruptible):**
- Gọi function component
- Chạy hooks (`useState`, `useMemo`, `useCallback`, ...)
- Tạo React Element tree
- Diff elements → gắn effect flags lên fibers
- **Không side effects** (chỉ là JS objects + data)
- Có thể bị pause, resume, abort

**Commit Phase (Synchronous):**
- Apply DOM mutations (insertions, updates, deletions)
- Update refs (`ref.current = node`)
- Run layout effects (`useLayoutEffect`)
- Browser paint
- Run passive effects (`useEffect`)
- **Không thể interrupt** (phải atomic để UI không hỏng)

#### C. Hệ quả / Pitfalls

- **Tại sao phải tách?**
  - Nếu pause giữa DOM update → UI hỏng (half-updated state)
  - Nếu abort render nhưng đã mutate DOM → inconsistent state
  - Tách ra:
    - Render phase an toàn để interrupt (chỉ là JS, chưa đụng DOM)
    - Commit phase an toàn cho UI (atomic, không bị pause)
- **Pitfall**:
  - Side effects trong render phase → có thể chạy nhiều lần hoặc không chạy (nếu render aborted)
  - Assume render chỉ chạy 1 lần → logic sai trong concurrent mode

#### D. Khi nào quan trọng

- Khi viết code:
  - Render phase: pure, no side effects
  - Commit phase / effects: side effects OK
- Khi debug:
  - Nếu side effect chạy nhiều lần → đang ở sai chỗ (render thay vì effect)
  - Nếu UI không update → render có thể bailout hoặc abort trước commit

---

### TL;DR / Bullet summary

- Render phase: tính toán, interruptible, pure, no side effects
- Commit phase: apply DOM, synchronous, atomic
- Render có thể bị discard, commit phải hoàn thành trọn vẹn
- Side effects chỉ trong commit/effects, không trong render

---

## 7. Priority Scheduling

### Deep explanation

#### A. Bản chất

- Không phải mọi update đều ngang nhau về mức độ urgent.
- React phân loại updates theo **priority levels** và schedule accordingly.

#### B. Cơ chế

**Priority Levels:**
- **High (Synchronous)**: user input (click, typing, controlled input updates)
- **Normal**: network responses, state updates từ async code
- **Low (Deferred)**: offscreen content, analytics, non-urgent UI updates

**Mechanism:**
- High priority có thể **interrupt** low priority render đang chạy
- React pause low priority work, xử lý high priority ngay lập tức, sau đó resume (hoặc restart) low priority work

#### C. Hệ quả / Pitfalls

- Nếu không có priority:
  - Heavy render có thể block user input → UI lag
  - Mọi update cạnh tranh nhau → không guarantee responsiveness
- Với priority:
  - User typing luôn được ưu tiên → UI luôn responsive
  - Heavy updates có thể defer → không block critical path

#### D. Khi nào có giá trị

- Khi có concurrent features (React 18+):
  - Dùng `startTransition` để mark updates là non-urgent
  - React tự động defer non-urgent updates nếu có urgent updates
- Ví dụ:
  ```javascript
  import { useTransition } from 'react';
  
  const [isPending, startTransition] = useTransition();
  
  startTransition(() => {
    setSearchQuery(input);  // Non-urgent
  });
  // Typing vẫn responsive, không chờ searchQuery update xong
  ```

---

### TL;DR / Bullet summary

- Priority scheduling: urgent updates trước, defer non-urgent
- High priority có thể interrupt low priority render
- Dùng transitions để mark non-urgent updates
- Kết quả: UI luôn responsive

---

## 8. Time Slicing

### Deep explanation

#### A. Bản chất

- Time slicing là kỹ thuật chia render work thành **small chunks** và yield control cho browser định kỳ.
- Mục tiêu: tránh chiếm main thread quá lâu, cho browser cơ hội paint và handle events.

#### B. Cơ chế

- React cố gắng làm việc trong **frame budget** (~5-16ms tùy trình duyệt)
- Sau mỗi chunk, React yield control:
  - Browser có thể paint frame
  - Browser có thể handle user events
- Sau đó React resume work
- Flow: `Work chunk → Yield → Paint/Events → Resume → Next chunk → ...`

#### C. Hệ quả / Pitfalls

- **Pitfall**: Assume "5ms per chunk" hoặc thời gian cố định
  - **Sai**. React không guarantee thời gian cụ thể.
  - Time slicing dựa trên **frame budget động** và scheduler heuristics
- Time slicing **không làm render nhanh hơn** (có thể chậm hơn do overhead của yielding)
- Time slicing làm UI **responsive hơn** (không freeze)

#### D. Khi nào có giá trị

- Khi có heavy renders:
  - Không có time slicing: render 100ms liền → UI freeze 100ms
  - Có time slicing: render chia thành chunks, yield giữa chừng → UI vẫn responsive
- Trade-off:
  - Tăng total render time (do overhead)
  - Giảm perceived latency (UI không đơ)

---

### TL;DR / Bullet summary

- Time slicing chia work thành chunks, yield cho browser
- Không làm render nhanh hơn, nhưng làm UI responsive hơn
- Không guarantee thời gian cố định
- Trade-off: overhead vs responsiveness

---

## 9. Concurrent Features (React 18+)

### Deep explanation

#### A. Bản chất

- Fiber là nền tảng kỹ thuật cho **concurrent rendering**.
- Concurrent rendering cho phép React **prepare nhiều versions của UI đồng thời** (vẫn trên single thread).

#### B. Cơ chế

**Enabled Features:**
- **Concurrent rendering**: render nhiều WIP trees, chỉ commit 1 kết quả cuối cùng
- **Suspense**: "pause" rendering subtree để đợi data, show fallback UI
- **Transitions**: mark updates là non-urgent, cho phép interrupt

**Key Mental Model:**
- Có thể có **nhiều render attempts** (nhiều WIP trees được tạo)
- React **chỉ commit 1 kết quả** (WIP tree cuối cùng)
- Renders bị discard **không ảnh hưởng UI hiện tại** (current tree không đổi)

#### C. Hệ quả / Pitfalls

- **Pitfall**: Concurrent ≠ async UI
  - **Sai**. UI vẫn update synchronously (commit phase vẫn sync).
  - Concurrent = **interruptible render**, không phải async commit
- Render có thể chạy nhiều lần:
  - Low-priority render bắt đầu
  - High-priority update xảy ra → interrupt
  - React discard low-priority WIP, start lại với high-priority
- Code implications:
  - Render phải pure (vì có thể chạy nhiều lần)
  - Effects phải có cleanup (vì component có thể unmount/remount)

#### D. Khi nào có giá trị

- Khi build complex UIs:
  - Heavy data fetching + rendering
  - Dùng Suspense để tránh loading waterfall
  - Dùng Transitions để keep UI responsive khi có expensive updates
- Ví dụ:
  - Search: typing (urgent) + filter results (non-urgent via transition)
  - Tabs: click tab (urgent) + load tab content (suspend)

---

### TL;DR / Bullet summary

- Concurrent features: Suspense, Transitions, concurrent rendering
- Mental model: nhiều render attempts, chỉ commit 1
- Concurrent = interruptible render, không phải async UI
- Render phải pure vì có thể chạy nhiều lần

---

## FINAL SUMMARY — PART I

### Core Rules for React Reconciliation

1. **Render phase phải pure**
   - Idempotent, restartable, abortable
   - Không side effects
   - Có thể chạy nhiều lần hoặc không chạy

2. **Commit phase phải atomic**
   - Synchronous, không interrupt
   - Apply DOM mutations + run effects
   - Chỉ chạy nếu render thành công

3. **Identity (key + type) quyết định reuse**
   - Cùng key + type → reuse component instance (giữ state, DOM)
   - Khác key hoặc type → unmount old, mount new (reset state)

4. **Fiber = interruptible reconciliation**
   - Chia work thành units
   - Pause/resume/abort based on priority
   - Enable concurrent features
   - Vẫn single thread, không phải multi-threading

5. **2-tree architecture bảo vệ UI stability**
   - Current tree = committed UI
   - WIP tree = render attempt (có thể discard)
   - Không mutate current → UI luôn consistent

6. **Virtual DOM là representation, không phải magic**
   - Performance đến từ: diff + batching + commit separation
   - Không phải vì "Virtual DOM nhanh hơn Real DOM"

---

## 10. Diffing Algorithm — Heuristic O(n)

### Fact

* Tree diff “hoàn hảo” trong trường hợp tổng quát rất đắt (naive có thể cực cao).
* React chọn **heuristic** để trade-off: **predictable + nhanh** thay vì perfect.

---

### Mechanism

React diff dựa trên 2 rule chính (heuristic):

#### Rule 1 — Different type ⇒ Replace subtree

* Nếu **type khác nhau**, React coi như subtree đã đổi.
* Hành vi: **unmount subtree cũ → mount subtree mới**
* Hệ quả: state trong subtree **reset**.

```jsx
// Old
<div><Counter /></div>

// New
<span><Counter /></span>
// div -> span => replace subtree => Counter remount => state reset
```

#### Rule 2 — Same type ⇒ Reuse node, update props

* Nếu **type giống nhau**, React **reuse DOM node** hiện tại.
* Chỉ update props khác biệt.

```jsx
// Old
<div className="old" title="hello" />

// New
<div className="new" title="hello" />
// reuse <div>, update className only
```

---

### Pitfall

* ❌ “React diff sâu mọi thứ”
  → Không. Type đổi là coi như đổi tree.
* ❌ “Đổi wrapper không ảnh hưởng state con”
  → Có thể **reset state** vì remount.

---

### Practice

* Nếu muốn giữ state:

  * Giữ **type ổn định** ở các node bao quanh.
* Nếu muốn reset state có chủ đích:

  * Đổi type (ít dùng) hoặc dùng **key** (phổ biến hơn, kiểm soát tốt hơn).

---

### Verify / Debug

* Thêm log trong `useEffect(() => { ...; return cleanup })` của child:

  * Nếu wrapper type đổi → bạn sẽ thấy cleanup + mount lại.
* Quan sát React DevTools:

  * Component bị unmount/mount lại thay vì update.

---

### Takeaways

* Diffing của React là **heuristic**.
* **Type** là tín hiệu mạnh nhất quyết định reuse vs remount.
* Type đổi ⇒ subtree remount ⇒ state reset.

---

## 11. Key Prop — Identity của Element trong List

### Fact

* `key` là **identity** React dùng để match children giữa các lần render.
* React **không dùng content** để nhận diện element.
* Không có `key` → React fallback match theo **position (index)**.

---

### Mechanism

Khi reconcile array children:

#### Case A — Có `key`

* React match phần tử mới với phần tử cũ bằng **key**.
* Nếu **key + type khớp**:

  * reuse component instance
  * giữ nguyên state / refs / DOM

#### Case B — Không `key`

* React match theo **thứ tự trong mảng** (`i = 0,1,2...`)
* Dễ sai khi list có insert/delete/reorder.

---

### Pitfall

#### 1) Không key + list dynamic ⇒ “state đội lốt”

```jsx
items.map(item => <Item>{item}</Item>) // ❌ no key
```

Xóa phần tử giữa:

* React compare theo position
* Item “C” có thể reuse fiber của “B”
  → state/ref/effect của B bị gắn sang C.

#### 2) Dùng index làm key ⇒ bug khi insert/delete/reorder

```jsx
items.map((item, i) => <Item key={i} />) // ❌ index key
```

* Insert đầu list → tất cả key đổi → reuse sai hàng loạt
* Reorder → state nhảy / focus mất

---

### Practice

✅ Key phải:

* **unique**
* **stable theo data**
* **không phụ thuộc position**

```jsx
items.map(item => <Item key={item.id} item={item} />)
```

✅ Chỉ dùng index key nếu list **never changes**:

* không insert
* không delete
* không reorder

---

### Verify / Debug

Khi gặp bug kiểu: focus nhảy / input reset / state lẫn

1. List có dynamic không (insert/delete/reorder)?
2. Key đang là gì (id hay index)?
3. Log mount/unmount của item:

```jsx
useEffect(() => {
  console.log("mount", id);
  return () => console.log("unmount", id);
}, [id]);
```

* Nếu reorder mà mount/unmount bất thường → key sai.

---

### Takeaways

* `key` = identity, không phải “để hết warning”.
* List dynamic: **key theo data**, không theo index.
* Key quyết định **reuse vs remount**.

---

## 12. Key as a Control Tool (Beyond Lists)

### Fact

* `key` không chỉ dùng cho list.
* `key` là “núm điều khiển” cách React match element **giữa các lần render**.

---

### Mechanism

* Same type nhưng **key khác** ⇒ React coi là **element khác** ⇒ remount.

---

### Practice

#### 1) Force reset state khi context đổi (route/user)

```jsx
function UserProfile({ userId }) {
  return <ProfileForm key={userId} userId={userId} />;
}
```

* userId đổi ⇒ remount ⇒ form reset đúng user.

#### 2) Reset modal content theo mode

```jsx
function Modal({ contentType }) {
  return (
    <ModalContent key={contentType}>
      {contentType === "login" ? <LoginForm /> : <SignupForm />}
    </ModalContent>
  );
}
```

---

### Pitfall

* Lạm dụng key để “fix bug” có thể gây:

  * remount nhiều
  * mất state không mong muốn
  * performance cost

---

### Verify / Debug

* Nếu dùng key để reset, hãy đảm bảo:

  * Bạn **muốn** cleanup/mount lại effects
  * Bạn **chấp nhận** mất state

---

### Takeaways

* `key` là tool để:

  * giữ identity đúng trong list
  * chủ động reset lifecycle khi cần

---

## 13. Reconciliation Process — Summary

### Deep explanation

#### A. Bản chất

- Reconciliation có thể tổng kết thành 3 bước chính: compare root, compare children, component reconciliation.
- Mỗi bước dựa vào **type + key** để quyết định reuse hay remount.

#### B. Cơ chế

**Step 1 — Compare root:**
- Type khác → replace tree
- Type giống → update props, recurse vào children

**Step 2 — Compare children:**
- Không có key → so sánh theo order (position)
- Có key → match by key

**Step 3 — Component reconciliation:**
- Same type → update (preserve state)
- Different type → remount (reset state)

#### C. Hệ quả / Pitfalls

- **Type + Key** là 2 tín hiệu mạnh nhất.
- Trace được 2 thứ này → debug được phần lớn bug reconciliation.
- Không hiểu mechanism → dễ bị bugs: state reset bất ngờ, focus nhảy, input mất giá trị.

---

### TL;DR / Bullet summary

- Reconciliation = 3 bước: root → children → component
- Type + Key quyết định reuse vs remount
- Hiểu đúng để debug reconciliation bugs

---

## FINAL SUMMARY — PART I (Updated)

### Core Rules for React Reconciliation & Diffing

1. **Render phase phải pure**
   - Idempotent, restartable, abortable
   - Không side effects
   - Có thể chạy nhiều lần hoặc không chạy

2. **Commit phase phải atomic**
   - Synchronous, không interrupt
   - Apply DOM mutations + run effects
   - Chỉ chạy nếu render thành công

3. **Identity (key + type) quyết định reuse**
   - Cùng key + type → reuse component instance (giữ state, DOM)
   - Khác key hoặc type → unmount old, mount new (reset state)
   - Key phải: unique, stable theo data, không phụ thuộc position

4. **Diffing algorithm là heuristic O(n)**
   - Type khác → replace subtree (không diff sâu)
   - Type giống → reuse node, update props
   - Trade-off: predictable + nhanh thay vì perfect diff

5. **Fiber = interruptible reconciliation**
   - Chia work thành units
   - Pause/resume/abort based on priority
   - Enable concurrent features
   - Vẫn single thread, không phải multi-threading

6. **2-tree architecture bảo vệ UI stability**
   - Current tree = committed UI
   - WIP tree = render attempt (có thể discard)
   - Không mutate current → UI luôn consistent

7. **Virtual DOM là representation, không phải magic**
   - Performance đến từ: diff + batching + commit separation
   - Không phải vì "Virtual DOM nhanh hơn Real DOM"

8. **Key best practices**
   - List dynamic: key theo data (item.id), không dùng index
   - Key để control: force reset state khi context đổi
   - Tránh lạm dụng → remount không cần thiết

---

# II. RENDERING LIFECYCLE

---

### Overall strategy (thứ tự ưu tiên)

- React update **không phải** "render → DOM update ngay"
- 3 layer: **Schedule → Render → Commit**
- Render phase: pure, có thể repeat/discard
- Commit phase: atomic, mới thay đổi UI
- Hiểu đúng = code concurrent-safe

---

### Key decision rules

- setState chỉ schedule, không guarantee render/commit
- Render: pure, idempotent, restartable, no side effects
- Commit: synchronous, atomic, apply DOM + effects
- Bailout xảy ra trước commit
- "Re-render" = thuật ngữ nói miệng, không chính xác

---

### Common misconceptions

- "setState → UI update ngay" → sai
- "Render = DOM update" → sai
- "Component render → effect chạy" → sai
- "Render chỉ 1 lần/update" → sai (concurrent)
- "Bailout là bug" → sai, là optimization

---

## 1. React Update Pipeline

### Deep explanation

#### A. Bản chất

- 3 layer độc lập: Schedule → Render → Commit
- Không phải update nào cũng đi hết 3 layer
- setState chỉ schedule, chưa render, chưa commit

#### B. Cơ chế

**Layer 1 — Schedule:**
- setState / props change / context change
- Mark Fiber dirty
- Chưa gọi component, chưa đụng DOM

**Layer 2 — Render:**
- Gọi function component, chạy hooks
- Tạo Element tree, reconcile
- Pure, idempotent, restartable, abortable
- Có thể chạy nhiều lần hoặc bị discard

**Layer 3 — Commit:**
- Apply DOM mutations
- Update refs, run effects
- Synchronous, atomic, không interrupt

#### C. Hệ quả / Pitfalls

- Side effects trong render → chạy nhiều lần hoặc không chạy
- Logic dựa "setState → UI đổi ngay" → race conditions
- Concurrent mode: render có thể abort → state inconsistency nếu dùng external mutable

#### D. Khi nào quan trọng

- Viết code: render pure, effects trong commit
- Debug: phân biệt layer nào bị bottleneck
- Optimize: target đúng layer (render vs commit)

---

### TL;DR / Bullet summary

- 3 layers: Schedule → Render → Commit
- Render: pure, có thể repeat/discard
- Commit: atomic, mới update UI
- setState không guarantee render hay commit

---

## 2. Render Attempt vs Commit

### Deep explanation

#### A. Bản chất

- Render attempt: gọi component để tính output
- Commit: apply output vào DOM
- Không phải mọi render đều commit

#### B. Cơ chế

Ví dụ:

```ts
setCount(prev => prev); // setState cùng value
```

Flow:
1. Schedule update
2. Render: so sánh `Object.is(prev, next)` → true
3. Bailout → skip commit
4. UI không đổi, effects không chạy

Concurrent example:
- Low priority render đang chạy
- High priority update → discard low priority
- Render bị abort, không commit

#### C. Hệ quả / Pitfalls

- Render chạy ≠ UI update
- Effect chỉ chạy sau commit
- Concurrent: render có thể repeat/discard
- Logic dựa "render = effect chạy" → sai

#### D. Khi nào quan trọng

- Debug: "render mà UI không đổi" → check bailout
- Debug: "effect không chạy" → check commit
- Concurrent: render repeat → console.log nhiều lần

---

### TL;DR / Bullet summary

- Render attempt ≠ Commit
- Bailout trước commit → UI không đổi
- Effects chỉ sau commit
- Concurrent: render có thể discard

---

## 3. Bailout & Object.is

### Deep explanation

#### A. Bản chất

- Bailout = skip render khi không có gì thay đổi
- So sánh bằng `Object.is` (reference equality)
- Optimization hợp lệ, không phải bug

#### B. Cơ chế

React bailout nếu:
- State: `Object.is(prev, next) === true`
- Props (với memo): tất cả props giống reference
- Context: value giống reference

`Object.is`:
- `Object.is({}, {})` → false (khác reference)
- `Object.is(obj, obj)` → true (cùng reference)

Flow bailout:
1. setState
2. So sánh state
3. TRUE → skip component, skip children, skip commit, skip effects
4. FALSE → render tiếp

#### C. Hệ quả / Pitfalls

Pitfall 1: Mutate rồi setState

```ts
state.count++; // mutate
setState(state); // cùng reference → bailout
```

Pitfall 2: Object literal trong props

```ts
<Child config={{ theme: 'dark' }} /> // mỗi render = object mới
```

Fix: `useMemo(() => ({ theme: 'dark' }), [])`

Pitfall 3: Inline functions

```ts
<Child onClick={() => {}} /> // mỗi render = function mới
```

Fix: `useCallback`

#### D. Khi nào quan trọng

- Object/array state → immutable update
- React.memo → stable props (useMemo, useCallback)
- Debug "setState không render" → check reference

---

### TL;DR / Bullet summary

- Bailout = reference equality (`Object.is`)
- Immutable state bắt buộc
- Stable props cho memo
- Bailout = optimization, không phải bug

---

## 4. Effects & Lifecycle

### Deep explanation

#### A. Bản chất

- Effect = reaction với committed UI
- Không phải lifecycle hook
- Dùng để sync state với external system

#### B. Cơ chế

**Timing:**

- `useLayoutEffect`: commit phase, trước paint, block paint
- `useEffect`: sau paint, async, không block

**Flow:**

```
Render → Commit bắt đầu → DOM mutations → useLayoutEffect
→ Paint → useEffect
```

**Cleanup:**

```ts
useEffect(() => {
  const sub = subscribe();
  return () => sub.unsubscribe(); // cleanup
}, [dep]);
```

Cleanup chạy:
- Trước effect tiếp (dependency đổi)
- Khi unmount

#### C. Hệ quả / Pitfalls

- Render bailout → effect không chạy
- Render abort → effect không chạy
- Concurrent: render repeat → cần cleanup tránh leak
- Logic quan trọng trong effect → không deterministic

#### D. Khi nào dùng / tránh

Dùng:
- useEffect: fetch, subscriptions, async work
- useLayoutEffect: DOM measurement, adjust layout

Tránh:
- Logic quan trọng chỉ trong effect
- Async trong useLayoutEffect (block paint)
- Effect không cleanup

---

### TL;DR / Bullet summary

- Effect chỉ sau commit
- useLayoutEffect: sync, block paint
- useEffect: async, không block
- Luôn cleanup

---

## 5. Automatic Batching (React 18)

### Deep explanation

#### A. Bản chất

- React 18 batch mọi nơi (events, promises, timers)
- React 17 chỉ batch trong React events
- Batching = gom nhiều setState → 1 render + commit

#### B. Cơ chế

React 17:

```ts
onClick(() => { setState1(); setState2(); }); // ✅ batched
fetch().then(() => { setState1(); setState2(); }); // ❌ 2 renders
```

React 18:

```ts
fetch().then(() => { setState1(); setState2(); }); // ✅ batched
```

Hệ quả:
- Không đảm bảo số lần render/commit
- React merge/drop/reorder updates

#### C. Hệ quả / Pitfalls

- Đếm render để verify logic → sai
- Assume setState chạy ngay → stale read

```ts
setState(1);
console.log(state); // ❌ vẫn cũ
```

Fix: dùng updater function

```ts
setState(prev => prev + 1);
```

#### D. Khi nào quan trọng

- Debug: "setState nhiều mà render 1 lần" → batching
- Migrate React 17: code dựa sync setState → break

---

### TL;DR / Bullet summary

- React 18: batch everywhere
- Không đảm bảo số lần render
- Dùng updater function

---

## 6. flushSync — Escape Hatch

### Deep explanation

#### A. Bản chất

- Force sync render + commit ngay
- Break batching và concurrent
- Last resort

#### B. Cơ chế

```ts
import { flushSync } from 'react-dom';

flushSync(() => setState(val));
// DOM đã update, có thể đọc ngay
```

#### C. Pitfalls

- Performance cost
- Break concurrent features
- ❌ Không dùng default

#### D. Khi nào dùng / tránh

Dùng:
- Third-party DOM manipulation
- Measurement cần DOM state ngay

Tránh:
- Default fix timing issues
- Trong render phase

---

### TL;DR / Bullet summary

- flushSync = force sync commit
- Break concurrent
- Last resort only

---

## FINAL SUMMARY — PART II

### Core Rules for Rendering Lifecycle

1. **3 layers riêng biệt**: Schedule → Render → Commit
2. **Bailout = optimization**: reference equality, immutable state
3. **Effects sau commit**: useLayoutEffect sync, useEffect async
4. **Batching everywhere** (React 18): không đảm bảo số renders
5. **flushSync = last resort**: break concurrent
6. **Render phải pure**: no side effects, concurrent-safe
7. **"Re-render" không chính xác**: dùng scheduled/rendered/committed

---

# IV. PERFORMANCE OPTIMIZATION

---

## 🧭 Index layer (đọc 2–3 phút để kích hoạt toàn bộ mental model)

### Overall strategy (thứ tự ưu tiên)

- **Prevent re-renders** > Optimize re-renders > Reduce bundle size
- Architecture & composition **quan trọng hơn** memoization
- **Đo trước – tối ưu sau**
- Không tối ưu nếu **không có bottleneck rõ ràng**

---

### Key decision rules

- Ưu tiên:
  - Move state down
  - Composition (`children as props`)
  - Context splitting
- Chỉ dùng memoization khi:
  - Có re-render cascade
  - Render cost > 1 frame (~16ms)
  - Props có thể giữ reference ổn định
- Không tin “expensive calculation” nếu **chưa đo**
- React render thường **đắt hơn JS calculation**

---

### Common performance smells

- Memo khắp nơi nhưng không dùng Profiler
- useMemo / useCallback dùng theo cảm giác
- Split code vào critical path
- Context lớn gây re-render toàn app
- Optimize component thay vì **optimize tree**

---

> 📌 Nếu phần Index này đủ gợi lại toàn bộ kiến thức trong đầu bạn  
> → **không cần đọc Deep notes bên dưới**.

---

## 🧠 Deep notes (đọc khi cần đào sâu / quên chi tiết)

---

## 1. React.memo

### Mental model

- `React.memo` ngăn **re-render**, không tối ưu logic
- Dựa trên **shallow comparison props**
- So sánh bằng `Object.is`
- Chỉ hiệu quả khi **downstream có bailout**

---

### Cơ chế hoạt động

- Parent re-render
- React.memo nhận props mới
- So sánh từng prop với props cũ
  - TẤT CẢ giống → skip re-render
  - BẤT KỲ khác → re-render

---

### Vì sao memo thường “không hiệu quả”

- Props là:
  - object literal
  - function inline
- Reference đổi mỗi render
- Memo bị bypass hoàn toàn

---

### Khi memo có giá trị thật

- Component render chậm (> 16ms)
- Component nằm sâu trong tree
- Re-render cascade rõ ràng trong Profiler
- Props có thể giữ reference ổn định

---

### Anti-pattern phổ biến

- Memo leaf component không có cascade
- Memo trước khi đo
- Memo để che giấu architecture kém

---

## 2. useMemo vs useCallback

### Mental model

- Cả hai đều nhằm **giữ reference stability**
- `useCallback` memoize **function reference**
- `useMemo` memoize **return value**
- Function trong argument **luôn bị recreate**
- React chỉ cache **reference**, không cache function body

---

### Cơ chế quan trọng (hay bị hiểu sai)

- Ở mỗi lần render:
  - Function được truyền vào `useCallback` / `useMemo` **luôn được tạo mới**
- React thực hiện memoization theo 2 bước:
  - So sánh dependency array bằng `Object.is`
  - Quyết định có **tái sử dụng reference cũ** hay không
- React:
  - Cache **reference**
  - Không cache function body hoặc execution

---

### Hệ quả trực tiếp

- Nếu dependencies **không thay đổi**:
  - React trả lại **reference đã cache**
- Nếu dependencies **thay đổi**:
  - React trả lại **reference mới**
- Logic bên trong function:
  - Vẫn bị recreate mỗi render
  - Nhưng có thể không được expose ra ngoài nếu reference cũ được dùng lại

---

### Khi nên dùng useMemo

- Khi calculation:
  - Thực sự expensive
  - Chiếm đáng kể render time
- Khi cần:
  - Giữ identity ổn định cho object / array
  - Tránh re-render downstream do reference thay đổi
- Khi:
  - Giá trị memo là dependency của hook khác

---

### Khi nên dùng useCallback

- Khi:
  - Truyền function vào component dùng `React.memo`
  - Function là dependency của `useEffect` / `useMemo`
- Khi:
  - Custom hook cần trả về function ổn định

---

### Anti-pattern phổ biến

- Dùng `useMemo` cho calculation rẻ
- Dùng `useCallback` cho mọi event handler
- Memoization khi:
  - Chưa đo render time
  - Chưa xác định bottleneck

---

### Rule of thumb

- Đo:
  - Thời gian calculation
  - Thời gian render component
- Chỉ optimize khi:
  - Calculation chiếm ~10% render cost trở lên
- Ưu tiên:
  - Architecture
  - Composition
  - State colocation
- Memoization là:
  - Bước tối ưu cuối, không phải bước đầu

---

## 3. Code Splitting

---

### Deep explanation

#### A. Bản chất

- Mục tiêu:
  - Giảm initial bundle size
  - Tăng tốc initial load
  - Cải thiện FCP / TTI
- Strategy:
  - Tách code thành chunks
  - Load on-demand theo route / interaction

#### B. Cơ chế

- User request page:
  - Browser tải main bundle (nhỏ hơn)
  - React render initial UI
- Khi user navigate / interact:
  - Browser tải thêm chunk tương ứng
  - React render component mới

#### C. Hệ quả / Pitfalls

- Too much splitting:
  - Nhiều chunk nhỏ → nhiều request → overhead tăng
- Split critical path:
  - Component xuất hiện ngay nhưng lại lazy → chậm initial UI
- Thiếu loading states:
  - User thấy blank / flicker
- Waterfall:
  - Load code xong mới fetch data → tổng latency lớn

#### D. Khi nào dùng / Khi nào tránh

- Nên dùng khi:
  - Route-level splitting (mặc định nên có)
  - Component nặng chỉ xuất hiện theo tương tác
  - Feature flags / admin panels ít dùng
- Tránh khi:
  - Component nằm trong critical path
  - Chunk quá nhỏ gây request overhead
  - Không có fallback UI phù hợp

---

### TL;DR / Bullet summary

- Code splitting giảm initial bundle
- Nên ưu tiên route-level splitting
- Tránh split critical path
- Luôn có loading state
- Cẩn thận waterfall và quá nhiều chunk

---

## 4. Profiler API

---

### Deep explanation

#### A. Bản chất

- Công cụ built-in để đo performance rendering
- Dùng được để:
  - đo render cost thật
  - so sánh trước/sau optimization
  - monitor trong production

#### B. Cơ chế

- Wrap component bằng `<Profiler>` với `onRender`
- Callback nhận metrics cho mỗi commit:
  - `phase`: mount | update
  - `actualDuration`: thời gian render thực tế
  - `baseDuration`: thời gian ước tính nếu render toàn subtree
  - `commitTime`: thời điểm commit

#### C. Hệ quả / Pitfalls

- Không hiểu metrics → kết luận sai
- Đo 1 lần → nhiễu cao, không đại diện
- Optimize không dựa trên baseline → không chứng minh được improvement
- Focus vào micro-optimization thay vì bottleneck lớn

#### D. Khi nào dùng / Khi nào tránh

- Nên dùng khi:
  - Cần identify slow component có số liệu
  - Cần A/B compare optimization
  - Cần production monitoring theo interaction
- Tránh khi:
  - Chỉ “cảm giác chậm” mà chưa record
  - Dùng metrics như sự thật tuyệt đối không có context

---

### TL;DR / Bullet summary

- Profiler API để đo render cost thật
- `actualDuration` = thời gian render thực tế
- `baseDuration` = ước tính nếu render full subtree
- Luôn đo before/after, không tối ưu theo cảm giác

---

## 5. DevTools Profiler

---

### Deep explanation

#### A. Bản chất

- DevTools Profiler giúp trả lời 2 câu hỏi:
  - Component nào render chậm?
  - Vì sao nó render?

#### B. Cơ chế

- Record một interaction
- Dùng 3 view chính:
  - Ranked: sort theo thời gian render
  - Flame graph: chiều rộng ~ render time
  - Component tree: render reason (props/state/parent)

#### C. Hệ quả / Pitfalls

- Chỉ nhìn “component chậm” mà bỏ qua “render cascade”
- Fix bằng memo hóa hàng loạt → code phức tạp nhưng không nhanh hơn
- Không kiểm tra “render reason” → tối ưu nhầm chỗ

#### D. Khi nào dùng / Khi nào tránh

- Nên dùng khi:
  - Có interaction cụ thể gây lag
  - Cần tìm bottleneck theo evidence
- Tránh khi:
  - Tối ưu mà không record lại để verify

---

### TL;DR / Bullet summary

- Ranked view để tìm bottleneck nhanh
- Component tree để biết “why rendered”
- Luôn record → fix → record lại để so sánh

---

## 6. Architecture-level optimizations

---

### Deep explanation

#### A. Bản chất

- Mục tiêu:
  - Giảm blast radius của state changes
  - Prevent re-renders trước khi memo hóa
- Priority:
  - Composition & state colocation > memoization

#### B. Cơ chế

- Move state down:
  - Đặt state gần nơi dùng nhất để tránh re-render toàn tree
- Children as props:
  - Truyền `children` để giữ subtree stable khi wrapper re-render
- Context splitting:
  - Tách context theo “data” vs “api/actions”
  - Giữ reference ổn định cho phần ít thay đổi

#### C. Hệ quả / Pitfalls

- State đặt quá cao:
  - Một thay đổi nhỏ gây cascade re-render
- Context gộp quá nhiều:
  - Mọi consumer re-render khi bất kỳ field đổi
- “Memoization-first”:
  - Che symptom thay vì sửa cause

#### D. Khi nào dùng / Khi nào tránh

- Nên dùng khi:
  - Thấy cascade re-render lặp lại
  - State thay đổi cục bộ nhưng gây render global
  - Context có nhiều consumers
- Tránh khi:
  - Phân tách quá mức làm code khó hiểu hơn lợi ích thực tế

---

### TL;DR / Bullet summary

- Prevent re-renders bằng architecture trước
- Move state down để giảm blast radius
- Children as props để giữ subtree stable
- Split context để tránh re-render toàn bộ consumers

---

## Mental Model Tổng Hợp

---

### Deep explanation

#### A. Bản chất

- Performance optimization là bài toán:
  - giảm work trên main thread
  - giảm re-render không cần thiết
  - giảm payload tải ban đầu
- Thứ tự ưu tiên:
  - Prevent > Optimize > Reduce bundle > Measure

#### B. Cơ chế

- Prevent:
  - composition, state colocation, context splitting
- Optimize:
  - React.memo, useMemo, useCallback
- Reduce bundle:
  - code splitting, lazy loading
- Measure:
  - DevTools Profiler, Profiler API, before/after comparison

#### C. Hệ quả / Pitfalls

- Optimize không đo → dễ tối ưu sai
- Memo hóa bừa → tăng complexity, không tăng performance
- Split bừa → request overhead, waterfall

#### D. Khi nào dùng / Khi nào tránh

- Nên:
  - bắt đầu bằng measurement
  - ưu tiên thay đổi có impact lớn
- Tránh:
  - micro-optimize theo cảm giác

---

### TL;DR / Bullet summary

- Prevent re-renders là ưu tiên số 1
- Memoization chỉ hữu ích khi downstream bailout và props stable
- Code splitting phải tránh critical path và waterfall
- Measure everything, luôn so sánh before/after

---

# V. CONTEXT API

> Deep Dive từ Principal Perspective

Được, để tôi phân tích Context API một cách sâu sắc, kết hợp insights từ document và kinh nghiệm thực tế.

---

## 1. When to Use Context

### **Concept cốt lõi:**
Context giải quyết **prop drilling** bằng cách tạo "đường hầm" xuyên qua component tree.

### **Mental Model:**

```
Normal prop passing:
App → Layout → Sidebar → Menu → MenuItem
  ↓      ↓        ↓       ↓        ↓
 user   user     user    user    user (5 levels!)

With Context:
App (Provider)
  ↓ (direct tunnel)
MenuItem (Consumer) - No intermediate props!
```

### **Khi NÊN dùng Context:**

**1. Data được dùng bởi nhiều components ở nhiều levels:**
```javascript
// Theme - dùng khắp nơi
const ThemeContext = createContext();

// Auth state - cần ở header, sidebar, protected routes
const AuthContext = createContext();

// i18n - mọi text component cần
const I18nContext = createContext();
```

**2. Shared state management (thay thế Redux cho app nhỏ):**
```javascript
// Shopping cart - accessed từ nhiều nơi
const CartContext = createContext();

// Notifications - toast có thể trigger từ đâu cũng được
const NotificationContext = createContext();
```

**3. Dependency injection pattern:**
```javascript
// Services/APIs - inject vào components cần
const ApiContext = createContext();
const LoggerContext = createContext();
```

### **Khi KHÔNG NÊN dùng Context:**

**1. Props chỉ đi qua 1-2 levels:**
```javascript
// ❌ Overkill
<UserContext.Provider value={user}>
  <Dashboard user={user}>
    <Profile user={user} />
  </Dashboard>
</UserContext.Provider>

// ✅ Just pass props
<Dashboard user={user}>
  <Profile user={user} />
</Dashboard>
```

**2. Data thay đổi rất thường xuyên:**
```javascript
// ❌ Bad: Mouse position, scroll position
const MouseContext = createContext();

// Every pixel move = all consumers re-render!
```

**3. Component composition có thể giải quyết:**
```javascript
// ❌ Context không cần thiết
const Modal = () => {
  const { isOpen } = useModalContext();
  if (!isOpen) return null;
  return <div>Modal</div>;
};

// ✅ Better: Props + composition
const App = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setIsOpen(true)} />
      {isOpen && <Modal />}
    </>
  );
};
```

---

---

## 2. Performance Implications

### **Critical Understanding từ Document:**

> "Context consumers will re-render when the value on the Provider changes. ALL of them will re-render, even if they don't use the part of the value that actually changed."

### **The Re-render Problem:**

```javascript
const MyContext = createContext();

const Provider = ({ children }) => {
  const [user, setUser] = useState({ name: 'John' });
  const [theme, setTheme] = useState('dark');
  
  // ⚠️ Object bị re-create mỗi render!
  const value = { user, theme, setUser, setTheme };
  
  return (
    <MyContext.Provider value={value}>
      {children}
    </MyContext.Provider>
  );
};

// Component này CHỈ dùng user
const UserDisplay = () => {
  const { user } = useContext(MyContext);
  // ❌ Nhưng sẽ re-render khi theme thay đổi!
  return <div>{user.name}</div>;
};
```

### **Why this happens:**

```
1. theme state changes
    ↓
2. Provider re-renders
    ↓
3. value object re-created (new reference!)
    ↓
4. React compares: oldValue !== newValue
    ↓
5. ALL consumers re-render (kể cả UserDisplay)
```

### **Solution 1: Memoize the value**

```javascript
const Provider = ({ children }) => {
  const [user, setUser] = useState({ name: 'John' });
  const [theme, setTheme] = useState('dark');
  
  // ✅ Memoize để giữ stable reference
  const value = useMemo(
    () => ({ user, theme, setUser, setTheme }),
    [user, theme] // Chỉ thay đổi khi cần
  );
  
  return (
    <MyContext.Provider value={value}>
      {children}
    </MyContext.Provider>
  );
};
```

**⚠️ Warning:** Memoization không giải quyết vấn đề "all consumers re-render". Nó chỉ prevent re-render khi **Provider component** bị re-render bởi parent.

### **The Real Problem - Cascading Re-renders:**

```javascript
const Layout = () => {
  const [scroll, setScroll] = useState(0);
  
  // ❌ Provider là child của Layout
  // Khi scroll changes → Provider re-renders → value re-created
  return (
    <NavigationProvider>
      <Sidebar />
      <MainContent />
    </NavigationProvider>
  );
};
```

**Solution:**
```javascript
// ✅ Move Provider ra ngoài, không bị ảnh hưởng bởi scroll state
const App = () => (
  <NavigationProvider>
    <Layout />
  </NavigationProvider>
);
```

---

---

## 3. Provider Composition

### **Pattern 1: Multiple Contexts (Separation of Concerns)**

```javascript
// ❌ God Context - everything in one
const AppContext = createContext();

const AppProvider = ({ children }) => {
  const [user, setUser] = useState();
  const [theme, setTheme] = useState();
  const [cart, setCart] = useState([]);
  const [notifications, setNotifications] = useState([]);
  
  // Any change = all consumers re-render!
  return (
    <AppContext.Provider value={{ 
      user, setUser, theme, setTheme, 
      cart, setCart, notifications, setNotifications 
    }}>
      {children}
    </AppContext.Provider>
  );
};

// ✅ Split into focused contexts
const UserProvider = ({ children }) => {
  const [user, setUser] = useState();
  const value = useMemo(() => ({ user, setUser }), [user]);
  return (
    <UserContext.Provider value={value}>
      {children}
    </UserContext.Provider>
  );
};

const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('dark');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
};

// Compose them
const App = () => (
  <UserProvider>
    <ThemeProvider>
      <CartProvider>
        <NotificationProvider>
          <AppContent />
        </NotificationProvider>
      </CartProvider>
    </ThemeProvider>
  </UserProvider>
);
```

**Benefits:**
- Consumers chỉ re-render khi context họ dùng thay đổi
- Easier testing (mock individual contexts)
- Better code organization

### **Pattern 2: Split Data vs API (từ Document)**

Đây là pattern CỰC KỲ QUAN TRỌNG nhưng ít người biết:

```javascript
// Context 1: Data that changes
const DataContext = createContext();

// Context 2: Stable API/callbacks
const ApiContext = createContext();

const Provider = ({ children }) => {
  const [data, setData] = useState();
  
  // ✅ API never changes - no dependencies!
  const api = useMemo(() => ({
    update: (newData) => setData(newData),
    delete: () => setData(null),
    refresh: () => fetchData()
  }), []); // Empty deps!
  
  // ✅ Data changes, but API doesn't
  const dataValue = useMemo(() => ({ data }), [data]);
  
  return (
    <DataContext.Provider value={dataValue}>
      <ApiContext.Provider value={api}>
        {children}
      </ApiContext.Provider>
    </DataContext.Provider>
  );
};

// Component CHỈ cần API - never re-renders!
const DeleteButton = () => {
  const { delete: deleteItem } = useContext(ApiContext);
  return <button onClick={deleteItem}>Delete</button>;
};

// Component cần data - re-renders when data changes
const DataDisplay = () => {
  const { data } = useContext(DataContext);
  return <div>{data}</div>;
};
```

**Why this is powerful:**
- 90% components chỉ cần API (buttons, forms) → Never re-render
- 10% components hiển thị data → Chỉ re-render khi cần

### **Pattern 3: useReducer + Split Contexts (Advanced)**

Document nhấn mạnh pattern này:

```javascript
const StateContext = createContext();
const DispatchContext = createContext();

const Provider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  // ✅ dispatch never changes (React guarantees)
  // No need to memoize!
  
  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  );
};

// Custom hooks for easy usage
const useState = () => {
  const context = useContext(StateContext);
  if (!context) throw new Error('Must be used within Provider');
  return context;
};

const useDispatch = () => {
  const context = useContext(DispatchContext);
  if (!context) throw new Error('Must be used within Provider');
  return context;
};

// Usage
const Component = () => {
  const dispatch = useDispatch(); // ✅ Never causes re-render
  const { user } = useState(); // ✅ Re-renders only when state changes
  
  return (
    <button onClick={() => dispatch({ type: 'UPDATE_USER' })}>
      Update
    </button>
  );
};
```

**Benefits:**
- dispatch is always stable → Components using it never re-render
- Reducer pattern = predictable state updates
- Easier to debug with Redux DevTools

---

---

## 4. Avoiding Re-renders

### **Technique 1: Context Selectors (HOC Pattern)**

Document giới thiệu trick CỰC HAY sử dụng HOC + React.memo:

```javascript
const Context = createContext();

// ✅ HOC wrapper with selector
const withContextSelector = (Component, selector) => {
  const Wrapper = (props) => {
    const contextValue = useContext(Context);
    const selectedValue = selector(contextValue);
    
    // ✅ Memo the actual component
    const MemoComponent = React.memo(Component);
    
    return <MemoComponent {...props} selected={selectedValue} />;
  };
  
  return Wrapper;
};

// Usage
const UserName = ({ selected }) => {
  return <div>{selected}</div>;
};

// ✅ Only re-renders when user.name changes
const UserNameWithContext = withContextSelector(
  UserName,
  (context) => context.user.name
);
```

**How it works:**
```
Context value changes
    ↓
Wrapper re-renders (có useContext)
    ↓
selector extracts value
    ↓
Compare: oldValue === newValue?
    ↓
If same: MemoComponent skips re-render!
If different: MemoComponent re-renders
```

### **Technique 2: Bail Out Pattern**

```javascript
const Context = createContext();

const useContextSelector = (selector) => {
  const context = useContext(Context);
  const selectedRef = useRef();
  
  const selected = selector(context);
  
  // ✅ Compare selected value, not entire context
  if (selectedRef.current !== selected) {
    selectedRef.current = selected;
  }
  
  return selectedRef.current;
};

// Usage - component chỉ re-render khi user.name changes
const Component = () => {
  const userName = useContextSelector(
    (ctx) => ctx.user.name
  );
  
  return <div>{userName}</div>;
};
```

**⚠️ Limitation:** Vẫn không prevent re-render hoàn toàn, nhưng giảm được impact.

### **Technique 3: Compound Components Pattern**

```javascript
// Provider cung cấp data, nhưng không dictate structure
const TabsProvider = ({ children }) => {
  const [activeTab, setActiveTab] = useState(0);
  
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  );
};

// Compound components
const TabList = ({ children }) => {
  // Only re-renders when needed
  return <div role="tablist">{children}</div>;
};

const Tab = ({ index, children }) => {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  return (
    <button onClick={() => setActiveTab(index)}>
      {children}
    </button>
  );
};

const TabPanel = ({ index, children }) => {
  const { activeTab } = useContext(TabsContext);
  if (activeTab !== index) return null;
  return <div>{children}</div>;
};

// Usage - flexible và performant
const MyTabs = () => (
  <TabsProvider>
    <TabList>
      <Tab index={0}>Tab 1</Tab>
      <Tab index={1}>Tab 2</Tab>
    </TabList>
    <TabPanel index={0}>Content 1</TabPanel>
    <TabPanel index={1}>Content 2</TabPanel>
  </TabsProvider>
);
```

### **Technique 4: Lazy Context Creation**

```javascript
// ❌ Context created even if not used
const ExpensiveContext = createContext(expensiveInitialValue());

// ✅ Lazy initialization
const ExpensiveContext = createContext();

const ExpensiveProvider = ({ children }) => {
  // Only compute when Provider actually mounts
  const value = useMemo(() => expensiveInitialValue(), []);
  
  return (
    <ExpensiveContext.Provider value={value}>
      {children}
    </ExpensiveContext.Provider>
  );
};
```

---

---

## 5. Advanced Patterns

### **1. Data Provider Pattern (Mini Caching Layer)**

```javascript
// Pattern để fetch data ở root, use ở leaf components
const CommentsDataProvider = ({ children }) => {
  const [comments, setComments] = useState();
  
  useEffect(() => {
    // ✅ Fetch triggered khi Provider mounts
    fetch('/api/comments')
      .then(r => r.json())
      .then(setComments);
  }, []);
  
  return (
    <CommentsContext.Provider value={comments}>
      {children}
    </CommentsContext.Provider>
  );
};

// Root level - trigger fetches in parallel
const App = () => (
  <UserDataProvider>
    <CommentsDataProvider>
      <PostsDataProvider>
        <AppContent />
      </PostsDataProvider>
    </CommentsDataProvider>
  </UserDataProvider>
);

// Deep in tree - no prop drilling
const CommentsList = () => {
  const comments = useComments(); // From context
  return comments.map(c => <Comment key={c.id} {...c} />);
};
```

**Benefits:**
- Parallel data fetching (no waterfall)
- No prop drilling
- Components tự lấy data cần thiết

### **2. Context + Local State Hybrid**

```javascript
// Global state in Context
const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('dark');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

// Component uses both global + local state
const ThemedModal = () => {
  const { theme } = useContext(ThemeContext); // Global
  const [isOpen, setIsOpen] = useState(false); // Local
  
  // ✅ Local state changes không affect context consumers
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open</button>
      {isOpen && (
        <Modal theme={theme}>
          <ModalContent />
        </Modal>
      )}
    </>
  );
};
```

**Rule of thumb:**
- Context: Data cần share across components
- Local state: UI state, ephemeral data

---

---

## 6. Performance Debugging Checklist

Khi Context app chậm, check theo thứ tự:

### **1. Check Provider re-renders:**
```javascript
const Provider = ({ children }) => {
  console.log('Provider rendered'); // Bao nhiêu lần?
  
  const value = useMemo(() => ({ /* ... */ }), [deps]);
  
  return (
    <Context.Provider value={value}>
      {children}
    </Context.Provider>
  );
};
```

**Red flags:**
- Provider log mỗi keystroke
- Provider re-render nhưng value không thay đổi
- Parent component của Provider có unnecessary state

### **2. Check value reference stability:**
```javascript
const Provider = ({ children }) => {
  const value = { data: 'test' }; // ❌ New object every render!
  
  useEffect(() => {
    console.log('Value changed'); // Bao nhiêu lần?
  }, [value]);
  
  return <Context.Provider value={value}>{children}</Context.Provider>;
};
```

**Fix:**
```javascript
const value = useMemo(() => ({ data: 'test' }), []); // ✅
```

### **3. Check consumer count:**
```javascript
// How many components use this context?
// More consumers = more re-renders when value changes

// Tool: React DevTools → Components → Context

// If >50 consumers → Consider splitting context
```

### **4. Measure impact:**
```javascript
<Profiler id="ContextArea" onRender={(id, phase, actualDuration) => {
  console.log(`${id} took ${actualDuration}ms`);
}}>
  <Context.Provider value={value}>
    <App />
  </Context.Provider>
</Profiler>
```

---

---

## 7. Real-World Example: Form Context

Đây là example tổng hợp tất cả patterns:

```javascript
// 1. Split data vs API
const FormStateContext = createContext();
const FormApiContext = createContext();

// 2. Use reducer for complex state
const formReducer = (state, action) => {
  switch (action.type) {
    case 'SET_FIELD':
      return { ...state, [action.field]: action.value };
    case 'SET_ERROR':
      return { ...state, errors: { ...state.errors, [action.field]: action.error } };
    case 'SUBMIT':
      return { ...state, isSubmitting: true };
    case 'SUBMIT_SUCCESS':
      return { ...state, isSubmitting: false };
    default:
      return state;
  }
};

const FormProvider = ({ children, onSubmit }) => {
  const [state, dispatch] = useReducer(formReducer, {
    values: {},
    errors: {},
    isSubmitting: false
  });
  
  // 3. Memoize API (stable reference)
  const api = useMemo(() => ({
    setField: (field, value) => 
      dispatch({ type: 'SET_FIELD', field, value }),
    
    setError: (field, error) => 
      dispatch({ type: 'SET_ERROR', field, error }),
    
    submit: async () => {
      dispatch({ type: 'SUBMIT' });
      try {
        await onSubmit(state.values);
        dispatch({ type: 'SUBMIT_SUCCESS' });
      } catch (error) {
        dispatch({ type: 'SET_ERROR', field: 'general', error });
      }
    }
  }), [onSubmit, state.values]); // API changes khi values changes
  
  return (
    <FormStateContext.Provider value={state}>
      <FormApiContext.Provider value={api}>
        {children}
      </FormApiContext.Provider>
    </FormStateContext.Provider>
  );
};

// 4. Custom hooks
const useFormState = () => {
  const context = useContext(FormStateContext);
  if (!context) throw new Error('useFormState must be used within FormProvider');
  return context;
};

const useFormApi = () => {
  const context = useContext(FormApiContext);
  if (!context) throw new Error('useFormApi must be used within FormProvider');
  return context;
};

// 5. Optimized field component
const FormField = React.memo(({ name, label }) => {
  const { values, errors } = useFormState();
  const { setField } = useFormApi();
  
  // ✅ Chỉ re-render khi value của field này changes
  // (nhưng vẫn phụ thuộc vào entire state object)
  
  return (
    <div>
      <label>{label}</label>
      <input
        value={values[name] || ''}
        onChange={(e) => setField(name, e.target.value)}
      />
      {errors[name] && <span>{errors[name]}</span>}
    </div>
  );
});

// 6. Submit button - only uses API
const SubmitButton = () => {
  const { isSubmitting } = useFormState();
  const { submit } = useFormApi();
  
  return (
    <button onClick={submit} disabled={isSubmitting}>
      {isSubmitting ? 'Submitting...' : 'Submit'}
    </button>
  );
};

// Usage
const MyForm = () => (
  <FormProvider onSubmit={handleSubmit}>
    <FormField name="email" label="Email" />
    <FormField name="password" label="Password" />
    <SubmitButton />
  </FormProvider>
);
```

---

## **Mental Model Tổng Hợp**

```
Context Decision Tree:

Need to share data across components?
    ├─ Data used by 1-2 levels? → Props
    └─ Data used by 3+ levels? → Consider Context
        ├─ Data changes frequently? → Maybe use state management lib
        └─ Data changes occasionally?
            ├─ Split into multiple focused contexts
            ├─ Separate data from API
            ├─ Memoize values
            └─ Use selectors/HOC for fine-grained subscriptions

Provider Placement:
    ├─ Place as high as needed, not higher
    ├─ Wrap only branches that need the data
    └─ Avoid wrapping entire app if possible

Performance:
    ├─ Measure first (Profiler)
    ├─ Identify bottlenecks (how many consumers?)
    ├─ Apply optimizations incrementally
    └─ Re-measure
```

---

## 9. Key Principles - Principal Level

1. **Context is NOT free:** Mỗi context consumer = potential re-render
2. **Composition > Context:** Nếu có thể giải quyết bằng props/children, đừng dùng Context
3. **Split aggressively:** Nhiều small contexts > một god context
4. **Memoize by default:** Value trong Provider phải được memoize
5. **Data/API separation:** Game changer cho performance
6. **Measure everything:** Profiler là bạn thân nhất

**Quote từ Document:**
> "Context has a bad reputation when it comes to re-renders... but Context can prevent unnecessary re-renders and significantly improve the performance of our apps as a result. When applied correctly and carefully, of course."

