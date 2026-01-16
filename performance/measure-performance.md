# PHẦN 1 — MEASURE PERFORMANCE
Performance bắt đầu từ cách bạn đo, không phải từ cách bạn tối ưu

## 1.1 Performance là gì? (định nghĩa đúng trước đã)

Trước khi nói về LCP, INP hay Lighthouse, bắt buộc phải thống nhất định nghĩa. Nếu không, toàn bộ phần sau sẽ tối ưu sai hướng.

### Định nghĩa chuẩn để dạy
Performance là mức độ hệ thống đáp ứng được kỳ vọng của người dùng, trong điều kiện không lý tưởng.

Có 3 ý cực kỳ quan trọng trong câu này:

1. **Kỳ vọng của người dùng** → không phải kỳ vọng của dev
2. **Đáp ứng được** → không chỉ nhanh, mà là phản hồi có ý nghĩa
3. **Điều kiện không lý tưởng** → máy yếu, mạng chậm, tab nhiều, CPU bận

👉 Nếu thiếu bất kỳ ý nào, bạn đang nói về benchmark, không phải performance.

### Sai lầm phổ biến của Senior FE
- "App load 2s là nhanh"
- "Lighthouse 90 là ổn"
- "Máy anh chạy mượt"

Tất cả đều sai về bản chất, vì:
- Người dùng không quan tâm số
- Người dùng quan tâm cảm giác kiểm soát được app hay không

---

## 1.2 Vì sao phải đo performance?

### Không phải để:
- Khoe điểm
- Làm đẹp báo cáo
- So sánh dev

### Mục đích duy nhất của measurement
**Measurement tồn tại để ra quyết định.**

Cụ thể là quyết định:
1. Có vấn đề thật không?
2. Vấn đề ở đâu?
3. Có đáng sửa không?
4. Sửa xong có tốt hơn không?

👉 Nếu metric không giúp trả lời 1 trong 4 câu hỏi trên, nó là noise.

---

## 1.3 3 lớp Measurement

Một trong những lỗi lớn nhất là trộn lẫn các loại metric
### 1️⃣ Lab Metrics – để debug

**Đặc điểm:**
- Môi trường kiểm soát
- Lặp lại được
- So sánh được

**Ví dụ:**
- Lighthouse
- DevTools Performance
- WebPageTest

**Dùng khi nào:**
- Tìm bottleneck
- Verify fix
- Phân tích kỹ thuật

❌ **Không bao giờ dùng để ưu tiên roadmap**

### 2️⃣ Field Metrics – để quyết định

**Đặc điểm:**
- User thật
- Device thật
- Network thật
- Hành vi thật

**Ví dụ:**
- Core Web Vitals ngoài production
- RUM
- 75th percentile

👉 Đây mới là performance ngoài đời.

**Sai lầm phổ biến:**
- Lấy Lighthouse thay cho Field
- Lấy average thay vì percentile

### 3️⃣ Business Metrics – để ưu tiên

Performance không tự tồn tại.

Một issue performance chỉ thực sự quan trọng khi:
- Ảnh hưởng conversion
- Ảnh hưởng retention
- Ảnh hưởng revenue
- Ảnh hưởng trust

👉 Principal FE phải dạy: **"Không gắn được với business → không ưu tiên"**

---

## 1.4 Core Web Vitals – hiểu đúng, không thần thánh hóa

CWV là baseline, không phải chân lý.

### CWV trả lời được gì?
- Page có ổn không?
- UX tổng thể có tệ không?

### CWV KHÔNG trả lời được gì?
- Feature này có chậm không?
- Flow này có gây bực không?
- Interaction cụ thể này có lag không?

👉 **Vì vậy: CWV cần nhưng không đủ**

**Sai lầm cực phổ biến:**
- Fix CWV xong → nghĩ là xong performance
- Đổ mọi vấn đề UX cho LCP / INP

👉 **CWV là symptom, không phải root cause**

---

## 1.5 Custom Performance Metrics – thứ phân biệt Senior và Principal

Nếu chỉ đo CWV:
- Bạn đang đo "page"
- Nhưng người dùng dùng "flow"

### Custom metric dùng để đo cái gì?
- Time từ action → feedback
- Time từ intent → result
- Time user mất quyền kiểm soát UI

**Ví dụ (không code):**
- Click search → thấy kết quả
- Click submit → thấy trạng thái
- Scroll → list render xong

👉 **Đây là nơi performance gắn trực tiếp với UX**

---

## 1.6 Long Task – bản chất thật sự của "app bị lag"

Rất nhiều dev nghĩ app chậm vì:
- Network
- Render
- React

**Thực tế:**
Phần lớn UX lag là do main thread bị chiếm quá lâu.

### Long Task là gì (bản chất)
- Một đoạn JS chiếm main thread > ~50ms
- Trong thời gian đó:
  - Không click được
  - Không scroll mượt
  - Không phản hồi

👉 Người dùng không biết "JS đang chạy"  
👉 Người dùng chỉ thấy app đơ

---

## 1.7 Performance Regression – thứ giết app từ từ

Performance hiếm khi:
- Tệ đột ngột
- Chết ngay

Nó thường:
- Tệ dần theo từng PR
- Không ai để ý
- Đến lúc user chửi mới biết

### Vì sao regression xảy ra?
- Không có baseline
- Không ai ownership
- Không ai chịu trách nhiệm

👉 **Principal FE phải coi performance là state có thể suy giảm, giống memory leak.**

---

## 1.8 Performance Budget – kiểm soát thay vì cầu nguyện

Budget không phải để:
- Hạn chế dev
- Làm khó team

**Budget để: Ngăn app chết dần**

### Concept cốt lõi:
- Không phải "cố nhanh"
- Mà là "không được vượt ngưỡng"

---

## 1.9 Dạy PHẦN 1 như thế nào cho người khác

Nếu bạn dạy phần này bằng:
- Tool
- Điểm số
- Checklist

→ Học viên sẽ optimize sai.

### Cách dạy đúng:
1. Bắt đầu bằng trải nghiệm user
2. Phân biệt loại metric
3. Chỉ ra metric nào dùng để làm gì
4. Nhấn mạnh: metric là công cụ, không phải mục tiêu

---

## 1.10 Chrome DevTools Performance Tab – Thực chiến Debug

### A. Anatomy of a Performance Recording

**Không phải chỉ "record và nhìn"**  
Mà phải hiểu từng phần của flame chart

### 🔍 Các vùng quan trọng phải đọc được:

**1. Main thread activity**
- Vàng = JavaScript execution
- Tím = Rendering & Layout
- Xanh lá = Painting
- Xám = System/Idle

**2. Network waterfall**
- Khi nào resource bắt đầu load
- Blocking time
- Priority

**3. Frames (FPS)**
- Vạch đỏ = dropped frame
- Mục tiêu: 60fps = 16.67ms/frame

**4. Long Tasks**
- Vạch đỏ góc trên
- Click vào xem call stack

### 📋 Workflow debug chuẩn:

```
1. Record user interaction (không phải full page load)
2. Tìm Long Task (vùng đỏ)
3. Click vào task → xem Bottom-Up tab
4. Identify hàm chiếm thời gian nhất
5. Click vào source → jump to code
```

### ⚠️ Sai lầm phổ biến:
- Record quá dài (>10s) → khó phân tích
- Không throttle CPU/Network → không reproduce được vấn đề user gặp
- Nhìn Summary tab thay vì Bottom-Up
- Không filter noise (extensions, third-party)

### 💡 Tips cho Principal FE:
Dùng User Timing API để đánh dấu custom event:
```javascript
performance.mark('search-start');
// ... your code
performance.mark('search-end');
performance.measure('search', 'search-start', 'search-end');
```
→ Sẽ hiện trên timeline → dễ correlate với code

---

### B. Đọc Flame Chart ở level "judgment"

Senior FE thường **nhận diện được màu**, nhưng **không đưa ra được kết luận đúng**.

#### 1️⃣ Phân biệt "CPU-bound" vs "IO-bound" issue

**CPU-bound (JS-heavy):**
- Main thread vàng kéo dài
- Network xong nhưng UI chưa phản hồi
- Long task xuất hiện sau interaction

**Cách nhận biết nhanh trong 30s:**
- Nhìn network waterfall → nếu xong sớm
- Nhưng LCP vẫn chậm
- Và main thread toàn màu vàng
→ Đây là CPU-bound

**Hướng xử lý:**
- Break task (setTimeout, scheduler)
- Reduce JS execution
- Defer non-critical logic
- Web Worker cho heavy computation

---

**IO-bound (Network / resource):**
- Network waterfall kéo dài trước render
- Main thread rảnh (nhiều xám) nhưng UI chưa có gì
- LCP bị đẩy bởi resource late

**Cách nhận biết nhanh:**
- LCP element đợi network
- Main thread idle
- Critical resource load chậm
→ Đây là IO-bound

**Hướng xử lý:**
- Preload critical resources
- SSR / streaming
- Resource hints (dns-prefetch, preconnect)
- Optimize resource priority

**Case study thực tế:**

```
❌ Sai lầm phổ biến:
LCP = 4s
→ Dev thấy JS heavy
→ Code split mọi thứ
→ LCP vẫn 4s (vì đang đợi font)

✅ Debug đúng:
1. Check network tab → font load sau 3.5s
2. Preload font
3. LCP giảm xuống 1.2s
```

👉 **Principal FE phải dạy senior phân loại trước khi fix.**

---

#### 2️⃣ Phân biệt "Slow task" vs "Bad timing task"

Không phải task nào lâu cũng là vấn đề.

**Case 1: Task lâu nhưng OK**
- Task 200ms chạy khi app idle
- User không tương tác
- Không block input
→ **KHÔNG CẦN FIX**

**Case 2: Task ngắn nhưng NGUY HIỂM**
- Task 80ms nhưng chạy đúng lúc:
  - User click button
  - User scroll list
  - User type input
→ **PHẢI FIX NGAY**

**Ví dụ cụ thể:**

```javascript
// ❌ Bad timing - block interaction
button.addEventListener('click', () => {
  // 100ms synchronous work
  processHeavyData();
  updateUI();
});

// ✅ Good timing - defer work
button.addEventListener('click', () => {
  updateUI(); // Instant feedback
  setTimeout(() => {
    processHeavyData();
  }, 0);
});
```

**Trong DevTools Performance:**
- Tìm user interaction marker (click/input)
- Xem main thread ngay sau đó
- Nếu bị block > 50ms → bad timing

👉 **Performance là timing, không phải duration thuần.**

---

### C. Correlate Flame Chart với User Intent

**Workflow bổ sung:**

```
1. Đặt câu hỏi: User đang cố làm gì?
   - Load page?
   - Submit form?
   - Filter list?
   - Navigate?

2. Xác định timeline tương ứng
   - User click vào đâu
   - Khi nào scroll
   - Khi nào type

3. Xem JS đang làm gì đúng thời điểm đó
   - Có đáp ứng intent không?
   - Có làm việc không liên quan không?
```

**Case study thực tế:**

```
User intent: Click "Add to cart"
Expected: Button feedback + cart badge update

DevTools shows:
- 200ms: Analytics tracking
- 150ms: Re-render entire product list
- 50ms: Update cart badge

Problem: User đợi 400ms cho 1 action cần 50ms
Fix: Defer analytics, memo product list
```

👉 **Không gắn với intent → debug dễ đi lạc hướng.**

---

## 1.11 Lighthouse – Đọc Report Đúng Cách

### ❌ Điểm số KHÔNG phải thứ quan trọng nhất
### ✅ Diagnostics mới là phần có giá trị

### 📊 Cấu trúc report phải hiểu:

**1. Metrics (6 chỉ số chính)**
- FCP, LCP, TBT, CLS, SI, TTI
- Weighted score → biết metric nào ảnh hưởng điểm nhất

**2. Opportunities (ưu tiên cao)**
- Có estimate savings
- Sắp xếp theo impact
→ **Fix từ trên xuống**

**3. Diagnostics (context)**
- Không có savings estimate
- Nhưng giải thích TẠI SAO chậm

**4. Passed Audits (bỏ qua được)**

---

### 🎯 Workflow dùng Lighthouse đúng:

```
1. Run 3 lần (không phải 1 lần)
2. Lấy median score
3. Ignore score, focus vào:
   - Opportunities với savings > 500ms
   - Main thread work > 2s
   - JavaScript execution time > 1s
4. Cross-reference với DevTools Performance
```

### ⚠️ Sai lầm khi dùng Lighthouse:
- Chạy trên máy mạnh → score ảo
- Chạy 1 lần rồi tin
- Chạy trên localhost (không có CDN, cache)
- Optimize để được 100 điểm thay vì fix real issue

### 🔧 Config đúng:
- **Desktop:** Throttling tắt (vì user có máy tốt)
- **Mobile:** Moto G4 + Slow 4G (default)
- **Clear storage** mỗi lần chạy

---

### A. Khi nào NÊN tin Lighthouse, khi nào KHÔNG

**NÊN tin khi:**
- So sánh before / after cùng environment
- Tìm quick wins (unused JS, render-blocking)
- Phát hiện obvious issue (image không optimize, blocking scripts)

**KHÔNG NÊN tin khi:**
- App có nhiều interaction phức tạp
- App phụ thuộc auth / personalization
- App heavy client-side logic (dashboard, tools)

**Ví dụ cụ thể:**

```
✅ Tin Lighthouse:
- Landing page tĩnh
- Blog post
- Marketing site

❌ Không tin Lighthouse:
- Gmail
- Figma
- Admin dashboard
```

👉 **Lighthouse ≠ UX truth**  
👉 **Lighthouse = static approximation**

**Vậy thì dùng gì thay thế khi không tin Lighthouse?**
- RUM (Real User Monitoring)
- Custom performance marks trong production
- Session replay tools (LogRocket, FullStory)
- Synthetic monitoring với authenticated flows

---

### B. Mapping Lighthouse metric → hành động

| Lighthouse signal | Ý nghĩa thật | Hướng điều tra | Tool tiếp theo |
|-------------------|--------------|----------------|----------------|
| High TBT (>600ms) | JS block main thread | Tìm Long Task cụ thể | DevTools Performance |
| Slow LCP (>4s) | Resource priority hoặc render delay | Check network waterfall | DevTools Network + Performance |
| Low SI (<70) | Progressive rendering kém | Kiểm tra SSR/skeleton | DevTools Performance (paint timing) |
| CLS cao (>0.25) | Layout instability | Tìm element shift | DevTools Performance (Layout Shift) |
| Large DOM (>1500 nodes) | Render performance | Virtualization hoặc pagination | React Profiler |

👉 **Senior cần bridge từ report → tool khác.**

---

### C. Anti-pattern nguy hiểm cần ghi rõ

**Anti-pattern 1: Optimize metric nhưng làm tệ UX**

```javascript
// ❌ Fix TBT bằng cách defer mọi thứ
button.addEventListener('click', async () => {
  await new Promise(resolve => setTimeout(resolve, 0));
  await new Promise(resolve => setTimeout(resolve, 0));
  processData(); // TBT thấp nhưng user đợi lâu hơn
});
```

**Anti-pattern 2: Fix CLS bằng cách hide content**

```css
/* ❌ CLS = 0 nhưng UX tệ */
img {
  visibility: hidden; /* Không shift nhưng user không thấy gì */
}
```

**Anti-pattern 3: Optimize LCP bằng fake content**

```html
<!-- ❌ LCP nhanh nhưng vô nghĩa -->
<div style="width:100%;height:400px;background:#eee"></div>
<!-- Real content load sau -->
```

**Ví dụ metric tốt hơn nhưng UX tệ hơn:**

1. **TBT giảm 50% nhưng INP tăng 2x:**
   - Defer mọi event handler
   - User click → phải đợi script load

2. **LCP cải thiện nhưng perceived performance tệ:**
   - Preload hero image
   - Nhưng critical CSS bị delay
   - User thấy image trước, chữ sau

3. **CLS = 0 nhưng jarring:**
   - Reserve space quá lớn
   - Content nhảy vào đột ngột (không smooth)

👉 **Không phải fix metric nào cũng improve UX.**

**Rule of thumb:**
- Fix metric → Test với real user
- Nếu user complain nhiều hơn → revert
- Metric là means, không phải end

---

## 1.12 Web Vitals – Thu Thập Data Từ Real Users

### 📡 Các cách đo CWV ngoài production:

#### 1️⃣ Google Search Console (miễn phí, passive)
- Data từ Chrome User Experience Report (CrUX)
- 28 days rolling average
- Group theo URL pattern

**⚠️ Hạn chế:**
- Delay 1-2 ngày
- Chỉ có Chrome users (không có Safari, Firefox)
- Cần đủ traffic (threshold ~1000 visits/month)

---

#### 2️⃣ web-vitals JavaScript library (khuyên dùng)

```javascript
import {onCLS, onINP, onLCP} from 'web-vitals';

onLCP(metric => {
  // Gửi lên analytics
  analytics.send({
    name: 'LCP',
    value: metric.value,
    id: metric.id,
    page: window.location.pathname,
    
    // Context quan trọng:
    connection: navigator.connection?.effectiveType,
    deviceMemory: navigator.deviceMemory,
    
    // Debug info
    element: metric.entries[0]?.element?.tagName,
    url: metric.entries[0]?.url
  });
});

onINP(metric => {
  analytics.send({
    name: 'INP',
    value: metric.value,
    // Interaction type
    interactionType: metric.entries[0]?.name,
    // Target element
    target: metric.entries[0]?.target?.tagName
  });
});
```

**Tại sao cần context (connection, deviceMemory)?**
- Để segment data
- Device memory 2GB vs 8GB → LCP khác biệt lớn
- 4G vs 3G → cần khác strategy

---

#### 3️⃣ RUM Tools (paid, nhưng đầy đủ nhất)
- SpeedCurve, Datadog RUM, New Relic
- Có thể segment theo:
  - Device type
  - Geography
  - User cohort
  - A/B test variant
  - Authenticated vs guest

---

### 📊 Phân tích data đúng cách:

**Rule #1: Không nhìn average → nhìn P75 (75th percentile)**

```
Tại sao P75?
- Google đánh giá CWV tại P75
- Average bị outlier làm sai lệch
- P75 đại diện cho "typical bad experience"
```

**Rule #2: Segment theo device type**

```
Không so sánh:
Desktop LCP: 1.2s
Mobile LCP: 3.5s

Phải so sánh:
Desktop LCP (this month): 1.2s vs last month: 1.1s
Mobile LCP (this month): 3.5s vs last month: 3.2s
```

**Rule #3: Segment theo page type**

```
Homepage ≠ Product page ≠ Checkout
Mỗi page có budget riêng
```

**Rule #4: Track theo thời gian (regression detection)**

```
Weekly trend:
Week 1: LCP 2.1s
Week 2: LCP 2.3s ⚠️
Week 3: LCP 2.8s 🚨
Week 4: LCP 3.2s 💀
```

---

### A. Phân tích chênh lệch P50 – P75 – P95

**Pattern recognition (rule-of-thumb):**

| Pattern | Nguyên nhân có thể | Action |
|---------|-------------------|---------|
| P50 tốt (1.5s), P75 xấu (4s), P95 rất xấu (8s) | Device/network yếu, low-end phone | Optimize cho mobile, test trên Moto G4 |
| P50/P75 tốt, P95 xấu (outliers) | Edge case, long session, memory leak | Session replay, monitor memory |
| Tất cả xấu (P50 > 3s) | Systemic issue, architecture problem | Refactor, SSR, code splitting |
| Chênh lệch lớn giữa P50 và P95 (>3x) | Không consistent, nhiều biến số | Kiểm tra A/B test, feature flags |

**Sample size cần bao nhiêu để tin pattern?**
- Minimum: 1000 samples
- Reliable: 5000+ samples
- Production-grade: 10000+ samples/week

**Ví dụ cụ thể:**

```
Case study: E-commerce checkout page

Data (1 tuần):
P50 LCP: 1.8s ✅
P75 LCP: 4.2s ⚠️
P95 LCP: 9.5s 🚨

Investigation:
- Segment by device memory:
  - 4GB+: P75 = 2.1s ✅
  - 2GB: P75 = 5.8s ❌
  - <2GB: P75 = 11s 💀

Root cause: Heavy JavaScript bundle
Fix: Code split payment libraries
Result: P75 giảm xuống 2.5s
```

---

### B. Correlate CWV với feature release

**Workflow regression detection:**

```
1. Mark release time trong analytics
   analytics.send({
     event: 'deployment',
     version: 'v2.3.1',
     timestamp: Date.now()
   });

2. Overlay CWV trend với deployment
   - Grafana / DataDog dashboard
   - Vertical line tại mỗi deploy

3. Detect regression window
   - LCP jump 20% sau deploy v2.3.1
   - Timeframe: 10:30 AM - 11:00 AM

4. Map về PR / feature
   - Git log --since="10:00" --until="11:00"
   - Identify suspect commits
   - Rollback hoặc hotfix
```

**Case study thực tế:**

```
Timeline:
- 9:00 AM: Deploy v2.5.0
- 9:30 AM: LCP P75 tăng từ 2.1s → 3.8s
- 10:00 AM: Alert trigger

Investigation:
- Compare bundle size: +120KB
- New feature: Image gallery component
- Root cause: Import entire swiper library

Fix options:
1. Rollback (immediate)
2. Dynamic import gallery (1 hour)
3. Lighter library (3 hours)

Decision: Option 2 (balance speed + UX)
```

👉 **Không làm bước này → không bao giờ fix được regression thật.**

---

### C. Khi nào KHÔNG cần tối ưu CWV

**Senior rất hay mắc bẫy "metric-driven panic".**

**Không cần optimize CWV khi:**

1. **Admin dashboard / Internal tools**
   - User = employee, có training
   - Device = company laptop (mạnh)
   - Network = office wifi (nhanh)
   → Focus vào functionality, không phải LCP

2. **Low-traffic page (<1000 visits/month)**
   - ROI thấp
   - Effort > impact

3. **Authenticated power-user app**
   - Gmail, Figma, IDE
   - User chấp nhận initial load chậm
   - Nhưng PHẢI nhanh sau khi loaded

4. **POC / MVP trong giai đoạn đầu**
   - Validation > optimization
   - Premature optimization

**Vậy thì đo gì thay CWV?**

```javascript
// Custom metrics cho admin dashboard
performance.measure('time-to-interactive-table');
performance.measure('filter-response-time');
performance.measure('export-duration');

// Gửi lên analytics
analytics.send({
  metric: 'filter-response-time',
  value: 150, // ms
  tableSize: 5000 // rows
});
```

👉 **Performance optimization phải có ROI.**

**Framework tính ROI nhanh:**

```
ROI = (Impact × Frequency × User value) / Effort

Example:
- Fix LCP từ 4s → 2s
- Impact: +5% conversion
- Frequency: 10000 visits/week
- User value: $50 average order
- Effort: 2 weeks engineer time

ROI = (5% × 10000 × $50) / (2 weeks × $X/week)
```

---

## 1.13 React Profiler – Debug React Performance

### 🎯 React Profiler trả lời 2 câu hỏi:
1. Component nào render lại không cần thiết?
2. Component nào render lâu?

### 📊 Cách dùng:

```
1. Mở React DevTools → tab Profiler
2. Click Record (blue circle)
3. Thực hiện interaction (click, type, etc)
4. Stop recording
5. Phân tích
```

---

### 🔍 Đọc Flamegraph:

- **Chiều ngang:** component tree (parent → child)
- **Chiều cao:** thời gian render (càng cao càng lâu)
- **Màu vàng/đỏ:** component render chậm (>12ms)
- **Màu xám:** component không render (được memo)

### ⚠️ Patterns cần phát hiện:

#### 1️⃣ Component render nhưng not committed (wasted render)

**Trong Profiler:**
- Component màu vàng
- Nhưng không có "committed" badge
→ **Render nhưng DOM không thay đổi**

**Nguyên nhân:**
```javascript
// ❌ Object/array reference thay đổi mỗi render
function Parent() {
  const config = { theme: 'dark' }; // New object mỗi lần
  return <Child config={config} />;
}

// ✅ Stable reference
function Parent() {
  const config = useMemo(() => ({ theme: 'dark' }), []);
  return <Child config={config} />;
}
```

**Fix pattern:**
- `useMemo` cho objects/arrays
- `useCallback` cho functions
- Move static data ra ngoài component

---

#### 2️⃣ Cascading renders (cha render → toàn bộ tree render)

**Trong Profiler:**
- Click vào 1 component
- Thấy toàn bộ children render cùng lúc
- Dù children không nhận props mới

**Ví dụ cụ thể:**
```javascript
// ❌ Parent state change → 100 children re-render
function Dashboard() {
  const [counter, setCounter] = useState(0);
  
  return (
    <div>
      <Counter value={counter} onClick={() => setCounter(c => c + 1)} />
      <HeavyChart data={staticData} /> {/* Re-render không cần thiết */}
      <HeavyTable data={staticData} /> {/* Re-render không cần thiết */}
      <HeavyMap data={staticData} /> {/* Re-render không cần thiết */}
    </div>
  );
}

// ✅ Isolate state
function Dashboard() {
  return (
    <div>
      <CounterWidget /> {/* State isolated ở đây */}
      <HeavyChart data={staticData} />
      <HeavyTable data={staticData} />
      <HeavyMap data={staticData} />
    </div>
  );
}
```

**Rule of thumb:**
- State xuống thấp nhất có thể trong component tree
- Không lift state up nếu không cần thiết

---

#### 3️⃣ Context hell (toàn bộ consumers render khi context thay đổi)

**Trong Profiler:**
- 1 action khiến hàng chục components render
- Tất cả đều consume cùng 1 context
- Dù chỉ dùng 1 field trong context

**Ví dụ:**
```javascript
// ❌ Context chứa nhiều values
const AppContext = createContext();

function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('dark');
  const [locale, setLocale] = useState('vi');
  
  // Mỗi lần 1 trong 3 values thay đổi → ALL consumers re-render
  return (
    <AppContext.Provider value={{ user, theme, locale }}>
      <Dashboard />
    </AppContext.Provider>
  );
}

// ✅ Split contexts
const UserContext = createContext();
const ThemeContext = createContext();
const LocaleContext = createContext();

// Components chỉ subscribe context cần thiết
```

**Advanced pattern: Context selector**
```javascript
// Chỉ re-render khi specific field thay đổi
function useContextSelector(context, selector) {
  const value = useContext(context);
  const selected = selector(value);
  
  return useSyncExternalStore(
    context.subscribe,
    () => selector(context.getSnapshot()),
    () => selector(context.getServerSnapshot())
  );
}

// Usage
const theme = useContextSelector(AppContext, state => state.theme);
// Chỉ re-render khi theme thay đổi, không care user/locale
```

---

#### 4️⃣ List key anti-pattern (key không stable → toàn bộ list re-mount)

**Trong Profiler:**
- List render lại
- Mỗi item có "Unmounted" rồi "Mounted"
- Thay vì "Updated"

**Nguyên nhân:**
```javascript
// ❌ Key thay đổi mỗi render
items.map((item, index) => (
  <Item key={index} {...item} /> // Index không stable khi thêm/xóa
))

// ❌ Key là random
items.map(item => (
  <Item key={Math.random()} {...item} /> // Mỗi render là key mới
))

// ✅ Stable unique key
items.map(item => (
  <Item key={item.id} {...item} />
))
```

**Impact:**
```
100 items × re-mount = 100 unmounts + 100 mounts
vs
100 items × update = 100 updates (nhanh hơn 3-5x)
```

---

### 📊 Ranked List Tab – Tìm nhanh component chậm nhất

**Workflow:**
```
1. Record interaction trong Profiler
2. Switch sang tab "Ranked"
3. Sort by "Render duration"
4. Component đầu tiên = slowest
```

**Metrics trong Ranked tab:**
- **Render duration:** Tổng thời gian render (bao gồm children)
- **Self time:** Thời gian render riêng component đó (không bao gồm children)

**Cách đọc:**
```
Component A:
- Render duration: 80ms
- Self time: 5ms
→ Vấn đề ở children, không phải A

Component B:
- Render duration: 50ms
- Self time: 48ms
→ Vấn đề ở chính B (logic nặng)
```

**Pattern recognition:**
```
High duration + Low self time:
→ Optimize children
→ Hoặc dùng React.memo cho children

High duration + High self time:
→ Component logic nặng
→ useMemo expensive calculations
→ defer non-critical work
```

---

### 🔗 Interactions & Commits Correlation

React Profiler gắn user interaction với React commits.

**Cách đọc:**
```
1. Trong Profiler, click vào flame chart
2. Xem "Interactions" ở sidebar
3. Một interaction có thể gây nhiều commits
```

**Pattern đáng ngờ: Commit storm**
```
User click button
→ Commit 1: Button state update (10ms)
→ Commit 2: Analytics tracking (5ms)
→ Commit 3: Form validation (15ms)
→ Commit 4: UI feedback (8ms)
→ Commit 5: Side effect (12ms)

Total: 50ms across 5 commits
```

**Vấn đề:**
- React phải flush 5 lần
- Browser phải paint 5 lần
- User thấy UI jank

**Fix:**
```javascript
// ❌ Multiple state updates = multiple commits
function handleClick() {
  setState1(newValue1);
  setState2(newValue2);
  setState3(newValue3);
}

// ✅ Batch updates = 1 commit (React 18+)
function handleClick() {
  startTransition(() => {
    setState1(newValue1);
    setState2(newValue2);
    setState3(newValue3);
  });
}
```

**Advanced: Debug commit sequence**
```javascript
// Log mỗi commit
React.Profiler id="App" onRender={(id, phase, actualDuration) => {
  console.log(`Commit ${id}: ${phase} in ${actualDuration}ms`);
}}>
  <App />
</React.Profiler>
```

---

### 🧪 Profiler API – Đo Performance trong Production

**DevTools Profiler chỉ dùng được trong development.**  
Muốn đo production → dùng Profiler component.

```javascript
import { Profiler } from 'react';

function onRenderCallback(
  id,         // "id" của Profiler
  phase,      // "mount" hoặc "update"
  actualDuration,  // Thời gian render hiện tại
  baseDuration,    // Thời gian render ước tính không có memo
  startTime,       // Khi React bắt đầu render
  commitTime,      // Khi React commit
  interactions     // Set of interactions tracked
) {
  // Gửi lên analytics
  analytics.send({
    component: id,
    phase,
    duration: actualDuration,
    timestamp: commitTime
  });
}

// Wrap component cần đo
<Profiler id="Navigation" onRender={onRenderCallback}>
  <Navigation />
</Profiler>
```

**Use cases trong production:**
```javascript
// 1. Đo performance của critical path
<Profiler id="Checkout">
  <CheckoutFlow />
</Profiler>

// 2. A/B test performance
<Profiler id={experiment.variant} onRender={trackPerf}>
  {experiment.variant === 'A' ? <VariantA /> : <VariantB />}
</Profiler>

// 3. Detect regression
// Aggregate actualDuration theo version
// Alert khi P95 tăng > 20%
```

**⚠️ Performance overhead:**
- Profiler component có cost (~1-2% overhead)
- Chỉ wrap critical components
- Hoặc sample 10% users

**Aggregation strategy:**
```javascript
// Đừng gửi mỗi render (quá nhiều events)
let buffer = [];

function onRenderCallback(id, phase, actualDuration) {
  buffer.push({ id, phase, actualDuration, timestamp: Date.now() });
  
  // Gửi batch mỗi 10s
  if (buffer.length > 100) {
    analytics.sendBatch(buffer);
    buffer = [];
  }
}
```

---

### ❌ Khi nào KHÔNG dùng React Profiler

**1. Initial page load**
- Mọi component đều phải mount lần đầu
- Profiler chỉ show "everything renders" → không actionable
- **Dùng Chrome DevTools Performance thay thế**

**2. SSR / Server Components**
- React Profiler không chạy trên server
- Chỉ track client-side rendering
- **Dùng Server Timing API cho backend metrics**

**3. Non-React performance issues**
```
Scenarios:
- Network slow
- Large image decode
- Third-party script blocking
- Browser extension interference

→ React Profiler không thấy được
→ Dùng Chrome DevTools Performance
```

**4. Micro-optimizations không đáng kể**
```
Component render 2ms → optimize xuống 1ms
= Tiết kiệm 1ms
= User không cảm nhận được

→ Không đáng effort
→ Focus vào component render > 16ms (dưới 60fps)
```

---

### 🎯 Decision Tree: Dùng tool nào?

```
Issue: App feels slow
├─ Suspect: React re-renders
│  ├─ Development: React DevTools Profiler
│  └─ Production: Profiler API + analytics
│
├─ Suspect: JavaScript execution
│  └─ Chrome DevTools Performance (flame chart)
│
├─ Suspect: Network / Resources
│  └─ Chrome DevTools Network + Performance
│
└─ Suspect: Overall page load
   ├─ Lab: Lighthouse
   └─ Field: Web Vitals (CrUX / RUM)
```

**Workflow kết hợp:**
```
1. User complaint: "App chậm"
2. RUM data: INP cao (500ms)
3. Lighthouse: TBT cao (600ms)
4. Chrome DevTools Performance: Long Task trong JS
5. React Profiler: Component X render 200ms
6. Fix Component X
7. Verify với RUM: INP giảm xuống 150ms
```

---

### 💡 Tips & Tricks cho Principal FE

**1. Filter noise – Chỉ xem components quan trọng**
```
Profiler settings (gear icon):
☑ Hide commits below 1ms
☑ Record why each component rendered
```

**2. Permanent instrumentation**
```javascript
// Luôn wrap root để track regressions
if (process.env.NODE_ENV === 'production' && Math.random() < 0.1) {
  // 10% users
  root = (
    <Profiler id="App" onRender={sendToAnalytics}>
      <App />
    </Profiler>
  );
}
```

**3. Correlate với business metrics**
```javascript
onRenderCallback(id, phase, actualDuration) {
  if (id === 'ProductList' && actualDuration > 100) {
    // Slow render có correlation với bounce rate?
    analytics.send({
      event: 'slow_render',
      component: id,
      duration: actualDuration,
      sessionId: getSessionId()
    });
  }
}
```

**4. Compare before/after**
- Record baseline trước khi optimize
- Record lại sau khi optimize
- Compare 2 recordings side-by-side (Chrome DevTools cho phép)

---

## 1.14 Bundle Analysis – Tìm Bundle Bloat

### 🎯 Bundle Analysis trả lời câu hỏi gì?

**KHÔNG phải:**
- "Bundle bao nhiêu KB?"

**MÀ LÀ:**
- Tại sao bundle lớn?
- Code nào chiếm nhiều nhất?
- Đang ship code không dùng?
- Có duplicate dependencies không?
- Có thể giảm bao nhiêu một cách hiện thực?

👉 **Bundle size = #1 killer của LCP và TBT**

---

### 📖 Bản chất của Bundle Cost (không chỉ là file size)

Senior thường nghĩ: "Bundle 100KB = user download 100KB"

**SAI.**

Bundle có 3 loại cost:

#### 1️⃣ Download Cost (mọi người đều biết)
- Phụ thuộc network speed
- Gzip/Brotli giảm được 70-80%

```
main.js: 500KB raw
→ Gzip: 120KB
→ Brotli: 100KB
```

#### 2️⃣ Parse Cost (ít người quan tâm)
- Browser phải parse JS thành AST
- **Không nén được** (phải parse raw code)
- Mobile device chậm hơn desktop 2-5x

```
Benchmark (Moto G4):
- 100KB JS = ~100-200ms parse time
- 500KB JS = ~500-1000ms parse time
```

👉 **Parse cost > Download cost trên mobile**

#### 3️⃣ Execute Cost (nguy hiểm nhất)
- Chạy top-level code
- Initialize modules
- Block main thread

```javascript
// Mỗi import đều execute
import _ from 'lodash'; // Execute toàn bộ lodash
import moment from 'moment'; // Load all locales
import * from './utils'; // Run tất cả utils
```

**Case study thực tế:**
```
App bundle: 300KB (gzipped)

Timeline:
- Download: 800ms (4G)
- Parse: 400ms
- Execute: 1200ms ⚠️

Total: 2.4s chỉ để "ready"
User nhìn thấy page nhưng click không được
→ TTI = 2.4s
```

👉 **Bundle optimization không chỉ là giảm KB, mà là giảm cả 3 costs.**

---

### 🔧 Tool 1: webpack-bundle-analyzer

**Mục đích:**
- Visualize bundle composition
- Tìm largest modules
- Detect duplicate dependencies

**Setup:**
```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static', // Generate HTML report
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    })
  ]
};
```

**Với Vite:**
```javascript
// vite.config.js
import { visualizer } from 'rollup-plugin-visualizer';

export default {
  plugins: [
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true
    })
  ]
};
```

---

### 📊 Đọc Treemap của Bundle Analyzer

**Cấu trúc treemap:**
- Mỗi ô = 1 module
- **Size của ô = size của module**
- Màu sắc = package/directory

**Workflow đọc nhanh (30 giây):**
```
1. Nhìn ô lớn nhất → đó là suspect #1
2. Check: Có cần thiết không?
   - Nếu KHÔNG → remove/lazy load
   - Nếu CÓ → tìm alternative nhẹ hơn
3. Lặp lại cho top 5 modules lớn nhất
```

**Example phân tích:**

```
Treemap shows:
┌─────────────────────────────────────┐
│ moment.js (288KB)                   │ ⚠️ Lớn nhất
├─────────────────┬───────────────────┤
│ lodash (70KB)   │ chart.js (250KB)  │ 
├─────────────────┼───────────────────┤
│ react-dom       │ Your app code     │
│ (130KB)         │ (80KB)            │
└─────────────────┴───────────────────┘

Analysis:
1. moment.js: 288KB
   - Chỉ dùng .format() và .diff()
   - Action: Replace với date-fns/format (2KB)
   - Savings: 286KB

2. chart.js: 250KB
   - Chỉ dùng ở 1 page (Analytics)
   - Action: Dynamic import
   - Savings: 250KB (từ main bundle)

3. lodash: 70KB
   - Import toàn bộ nhưng chỉ dùng 5 functions
   - Action: Import specific functions
   - Savings: 60KB

Total potential savings: 596KB từ main bundle
```

---

### 🔍 Patterns phải phát hiện trong Bundle

#### Pattern 1: Duplicate Dependencies

**Triệu chứng trong treemap:**
- Thấy cùng 1 library xuất hiện nhiều lần
- Ví dụ: `lodash` và `lodash-es` cùng tồn tại

**Tại sao xảy ra:**
```
package.json: lodash@4.17.21
some-lib: depends on lodash@4.17.15
other-lib: depends on lodash-es@4.17.21

→ Webpack bundle tất cả 3 versions
```

**Fix:**
```javascript
// webpack.config.js
resolve: {
  alias: {
    'lodash-es': 'lodash'
  }
}

// Hoặc dùng dedupe plugin
```

**Impact:**
```
Before: 
- lodash: 70KB
- lodash@4.17.15: 70KB  
- lodash-es: 70KB
= 210KB

After:
- lodash: 70KB
= Savings: 140KB
```

---

#### Pattern 2: Unused Code (Dead Code)

**Triệu chứng:**
- Import toàn bộ library nhưng chỉ dùng 1-2 functions
- Import barrel file (`index.js` export nhiều thứ)

**Ví dụ cụ thể:**

```javascript
// ❌ Import toàn bộ
import _ from 'lodash';
const result = _.debounce(fn, 100);
// → Bundle 70KB, chỉ dùng 1 function (2KB)

// ✅ Import specific
import debounce from 'lodash/debounce';
// → Bundle 2KB

// ❌ Import barrel
import { Button, Icon, Modal } from '@/components';
// → Bundle toàn bộ components/index.js

// ✅ Import direct
import { Button } from '@/components/Button';
import { Icon } from '@/components/Icon';
// → Tree-shaking chỉ bundle Button + Icon
```

**Tools kiểm tra unused code:**
```bash
# webpack
npx webpack-bundle-analyzer --help

# Vite - check rollup output
npm run build -- --mode analyze
```

---

#### Pattern 3: Vendor Bundle Quá Lớn

**Triệu chứng:**
- `vendors.js` hoặc `chunk-vendors.js` > 300KB (gzipped > 100KB)
- Initial load phải đợi toàn bộ vendor code

**Vấn đề:**
```
Vendor bundle 500KB chứa:
- react + react-dom: 130KB
- UI library: 200KB
- date library: 50KB
- chart library: 120KB

User vào homepage:
→ Phải download 500KB
→ Nhưng homepage không dùng chart
```

**Fix: Split vendors theo route**

```javascript
// webpack.config.js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      // Core vendors (mọi page đều cần)
      defaultVendors: {
        test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
        name: 'vendors-core',
        priority: 10
      },
      
      // Heavy vendors (chỉ 1 số pages cần)
      chartVendors: {
        test: /[\\/]node_modules[\\/](chart\.js|d3)[\\/]/,
        name: 'vendors-charts',
        priority: 5
      }
    }
  }
}
```

**Hoặc: Dynamic import cho heavy libs**

```javascript
// ❌ Static import
import Chart from 'chart.js';

// ✅ Dynamic import
const Chart = await import('chart.js');
```

---

#### Pattern 4: Polyfills Không Cần Thiết

**Triệu chứng:**
- Bundle chứa polyfills cho browser cũ
- Nhưng >95% users dùng browser hiện đại

**Ví dụ:**
```javascript
// babel.config.js - BAD
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 0.25%, not dead" // Support tất cả browsers
    }]
  ]
}
// → Ship polyfills cho IE11

// GOOD
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 1%, last 2 versions, not dead" // Modern browsers only
      // Hoặc dựa vào browserslist
    }]
  ]
}
```

**Differential loading (advanced):**
```html
<!-- Modern bundle cho modern browsers -->
<script type="module" src="/app.modern.js"></script>

<!-- Legacy bundle cho old browsers -->
<script nomodule src="/app.legacy.js"></script>
```

Modern browsers chỉ load `app.modern.js` (nhỏ hơn 20-30%)

---

### 🔧 Tool 2: source-map-explorer

**Khác với webpack-bundle-analyzer:**
- Map bundle → source files
- Chính xác hơn (dựa vào source maps)
- Tốt hơn cho debugging cụ thể

**Setup:**
```bash
npm install -g source-map-explorer

# Build với source maps
npm run build

# Analyze
source-map-explorer 'dist/assets/*.js'
```

**Use case:**
```
webpack-bundle-analyzer: "lodash chiếm 70KB"
source-map-explorer: "lodash được import từ 12 files"

→ Dễ dàng tìm và fix all imports
```

---

### 🌐 Tool 3: bundlephobia.com

**Dùng TRƯỚC khi install package**

```bash
# Thay vì:
npm install moment

# Làm:
1. Vào bundlephobia.com/package/moment
2. Xem:
   - Size: 288KB (minified)
   - Gzip: 71KB
   - Dependencies: 0
   - Composition: 90% locale files
   
3. Check alternatives:
   - date-fns: 78KB → 13KB (specific imports)
   - dayjs: 6.5KB (toàn bộ)
   
4. Decision: Dùng date-fns hoặc dayjs
```

**Rule of thumb:**
```
Package > 50KB (minified) → cân nhắc kỹ
Package > 100KB → phải có lý do rất tốt
Package > 200KB → tìm alternative
```

---

### 📏 Bundle Metrics Quan Trọng

#### 1. Initial Bundle Size
- First paint cần load bao nhiêu JS
- Target: < 200KB (gzipped < 70KB)

#### 2. Total Bundle Size  
- Toàn bộ app (sau khi load hết chunks)
- Target: < 500KB (gzipped < 150KB)

#### 3. Chunk Count
- Quá nhiều chunks → network overhead
- Quá ít chunks → bundle bloat
- Sweet spot: 3-10 chunks cho typical app

#### 4. Duplicate Code %
```
Total bundle: 500KB
Duplicate: 30KB
→ 6% duplicate (acceptable)

Duplicate > 10% → có vấn đề
```

---

### 🎯 Performance Budget Enforcement

**Concept:**
Không chỉ "cố gắng giảm bundle"  
Mà là **"ngăn bundle tăng"**

#### Webpack Budget:
```javascript
// webpack.config.js
performance: {
  maxAssetSize: 244000, // 244KB (uncompressed)
  maxEntrypointSize: 244000,
  hints: 'error' // Build fail nếu vượt
}
```

#### Vite Budget:
```javascript
// vite.config.js
build: {
  chunkSizeWarningLimit: 500, // KB
  rollupOptions: {
    output: {
      manualChunks: {
        'vendor': ['react', 'react-dom'],
        'ui': ['@mui/material']
      }
    }
  }
}
```

#### bundlesize (CI Integration):
```json
// package.json
{
  "bundlesize": [
    {
      "path": "./dist/main.*.js",
      "maxSize": "100 kB"
    },
    {
      "path": "./dist/vendor.*.js",
      "maxSize": "150 kB"
    }
  ]
}
```

```bash
# CI pipeline
npm run build
npx bundlesize
# → Fail nếu vượt budget
```

---

### 📚 Case Study: E-commerce App Optimization

**Initial state:**
```
Analysis:
- main.js: 800KB (raw) / 210KB (gzip)
- LCP: 4.2s
- TBT: 1200ms
```

**Step 1: Bundle analysis**
```bash
npm run build
npx webpack-bundle-analyzer dist/stats.json
```

**Findings:**
```
Treemap shows:
1. moment.js: 288KB
   - Used: date formatting
   - Usage: Product listing, checkout
   
2. lodash: 70KB × 2 (duplicate!)
   - package.json: lodash@4.17.21
   - some-dep: lodash@4.17.15
   
3. chart.js: 250KB
   - Used: Only in /analytics page
   - Currently: In main bundle
   
4. @mui/material: 350KB
   - Used: 10 components
   - Currently: Importing from barrel
```

**Step 2: Optimization actions**

```javascript
// 1. Replace moment with date-fns
- import moment from 'moment';
+ import { format, differenceInDays } from 'date-fns';

// 2. Fix lodash duplicate
// webpack.config.js
resolve: {
  alias: {
    'lodash': require.resolve('lodash')
  }
}

// 3. Dynamic import chart.js
- import Chart from 'chart.js';
+ const Chart = await import('chart.js');

// 4. Fix MUI imports
- import { Button, TextField, Dialog } from '@mui/material';
+ import Button from '@mui/material/Button';
+ import TextField from '@mui/material/TextField';
```

**Step 3: Results**

```
After optimization:
- main.js: 280KB (raw) / 75KB (gzip) ✅
- vendors.js: 150KB / 45KB
- analytics.chunk.js: 250KB / 65KB (lazy loaded)

Metrics:
- LCP: 4.2s → 1.8s (-57%)
- TBT: 1200ms → 280ms (-77%)
- Initial bundle: 210KB → 75KB (-64%)

Business impact:
- Bounce rate: -15%
- Conversion rate: +8%
```

**Step 4: Prevent regression**

```json
// package.json
"bundlesize": [
  {
    "path": "./dist/main.*.js",
    "maxSize": "80 kB"
  }
]
```

```yaml
# .github/workflows/ci.yml
- name: Build
  run: npm run build
  
- name: Check bundle size
  run: npx bundlesize
```

👉 **Mỗi PR phải pass bundle size check hoặc bị reject**

---

### 💡 Advanced Techniques

#### 1. Tree Shaking Optimization

```javascript
// package.json
{
  "sideEffects": false // Enable aggressive tree-shaking
}

// Hoặc specific files
{
  "sideEffects": ["*.css", "*.scss"]
}
```

#### 2. Scope Hoisting (Webpack)

```javascript
// webpack.config.js
optimization: {
  concatenateModules: true // ModuleConcatenationPlugin
}

// Giảm IIFE wrappers → smaller bundle
```

#### 3. Code Splitting Strategies

```javascript
// Route-based splitting
const Home = lazy(() => import('./pages/Home'));
const Analytics = lazy(() => import('./pages/Analytics'));

// Component-based splitting
const HeavyChart = lazy(() => import('./components/HeavyChart'));

// Library splitting
if (needsPDF) {
  const jsPDF = await import('jspdf');
}
```

---

### ⚠️ Anti-patterns Nguy Hiểm

**1. Optimize quá mức → DX tệ**
```javascript
// ❌ Quá extreme
import debounce from 'lodash/debounce';
import throttle from 'lodash/throttle';
import cloneDeep from 'lodash/cloneDeep';
// → 50 imports riêng lẻ

// ✅ Balance
import { debounce, throttle, cloneDeep } from 'lodash-es';
// → Tree-shaking vẫn work, code clean hơn
```

**2. Code splitting mọi thứ**
```javascript
// ❌ Over-splitting
const Button = lazy(() => import('./Button')); // 2KB component
const Icon = lazy(() => import('./Icon'));     // 1KB component

// → Network overhead > savings
```

Rule: Chỉ split chunk **> 20KB**

**3. Dùng size làm metric duy nhất**
```javascript
// Smaller bundle KHÔNG phải lúc nào cũng tốt hơn

Option A: 
- Bundle: 100KB
- Parse time: 50ms
- No runtime cost

Option B:
- Bundle: 80KB  
- Parse time: 100ms (code xấu, nhiều eval)
- Runtime cost: 200ms

→ Option A tốt hơn dù lớn hơn
```

---

## 🎓 Tổng kết Bundle Analysis

### ✅ Phải làm:
1. **Setup bundle analyzer** trong project
2. **Chạy phân tích** sau mỗi major feature
3. **Set performance budget** trong CI
4. **Check bundlephobia** trước khi install package
5. **Track bundle trend** theo version

### ❌ Tránh:
1. Optimize bundle 1 lần rồi quên
2. Tin vào bundle size mà không đo real performance
3. Over-split hoặc under-split
4. Import từ barrel files
5. Ship polyfills không cần thiết

### 🎯 Decision Framework:

```
Khi nào optimize bundle?
├─ Initial bundle > 200KB (gzip > 70KB) → NGAY
├─ LCP > 2.5s và JS heavy → NGAY  
├─ TBT > 300ms → HIGH PRIORITY
└─ Bundle tăng > 20% so với tháng trước → INVESTIGATE

Optimize cái gì trước?
1. Largest modules (top 5)
2. Duplicate dependencies
3. Unused code (dead imports)
4. Heavy libs chỉ dùng ở 1-2 pages (lazy load)
5. Polyfills không cần thiết
```

---

## 1.15 INP (Interaction to Next Paint) – Metric Quan Trọng Nhất Cho Responsiveness

### 🎯 Tại sao INP quan trọng hơn FID?

**Timeline:**
- 2020: Google giới thiệu FID (First Input Delay)
- 2022: Google announce INP (Interaction to Next Paint)
- **March 2024: INP thay thế FID trong Core Web Vitals** ⚠️

**Vấn đề với FID (deprecated):**
```
FID chỉ đo FIRST interaction
→ Không phản ánh toàn bộ UX

Example:
Page load: Click button → 50ms (fast FID ✅)
After 5s: Click button → 800ms (slow, nhưng FID không bắt ❌)

→ FID không phát hiện được performance regression trong session
```

**INP giải quyết:**
```
INP đo ALL interactions trong toàn bộ page session
→ Lấy worst case (P98 - percentile 98)
→ Phản ánh trải nghiệm thật

Example:
User có 50 interactions:
- 45 interactions: < 100ms
- 3 interactions: 200-300ms
- 2 interactions: 600ms (worst)

INP = ~600ms (P98)
→ Phát hiện được vấn đề
```

👉 **INP = Most representative metric cho responsiveness**

---

### 📖 INP Budget (Google Recommendation)

```
Good INP:     < 200ms  ✅ Tốt
Needs Improvement: 200-500ms  ⚠️ Cần improve
Poor INP:     > 500ms  🚨 Tệ
```

**So sánh với thực tế:**
```
200ms:
- User cảm thấy instant
- Không có perceived lag

500ms:
- User nhận thấy delay
- Bắt đầu frustration

> 800ms:
- User nghĩ app bị đơ
- Click lại nhiều lần
- Hoặc bounce
```

---

### 🔬 Anatomy of INP – 3 Phases

INP không phải là "1 số đơn giản". Nó là tổng của 3 phases:

```
INP = Input Delay + Processing Time + Presentation Delay
```

#### Phase 1: Input Delay
**Thời gian từ user interaction → browser bắt đầu xử lý event handler**

```
User clicks button
    ↓
    ? ms waiting...  ← Input Delay
    ↓
Event handler starts
```

**Nguyên nhân Input Delay cao:**
- Main thread đang bận (Long Task)
- Browser đang parse/compile JS
- Third-party script đang chạy

**Ví dụ:**
```javascript
// Long Task đang chạy
for (let i = 0; i < 1000000; i++) {
  heavyComputation(); // Block main thread
}

// User click button ngay lúc này
// → Phải đợi Long Task xong
// → Input Delay cao
```

---

#### Phase 2: Processing Time
**Thời gian chạy event handler + React re-render**

```
Event handler starts
    ↓
    ? ms executing... ← Processing Time
    ↓
Event handler finishes
```

**Bao gồm:**
- Event handler code
- State updates (setState)
- React reconciliation
- Component re-renders

**Ví dụ:**
```javascript
// ❌ Heavy processing in event handler
function handleClick() {
  // 150ms: Filter 10000 items
  const filtered = items.filter(complexLogic);
  
  // 50ms: Sort
  const sorted = filtered.sort(heavySort);
  
  // 100ms: React re-render entire list
  setResults(sorted);
}
// → Processing Time = 300ms
```

---

#### Phase 3: Presentation Delay
**Thời gian từ React xong → browser paint lên screen**

```
React reconciliation done
    ↓
    ? ms rendering... ← Presentation Delay
    ↓
Visual feedback on screen
```

**Bao gồm:**
- Layout calculation
- Paint
- Composite
- Browser rendering pipeline

**Ví dụ:**
```javascript
// ❌ Force layout recalculation
function handleClick() {
  element.classList.add('active');
  
  // Force layout
  const height = element.offsetHeight; // 🚨
  
  anotherElement.style.height = height + 'px';
}
// → Presentation Delay cao vì layout thrashing
```

---

### 🔍 Debug INP Cao – Workflow Chuẩn

#### Step 1: Thu thập INP data

**Sử dụng web-vitals library:**
```javascript
import { onINP } from 'web-vitals';

onINP((metric) => {
  console.log('INP:', metric.value);
  console.log('Attribution:', metric.attribution);
  
  // Attribution cho biết:
  // - interactionTarget: Element nào bị chậm
  // - interactionType: 'pointer' | 'keyboard'
  // - loadState: 'loading' | 'dom-interactive' | 'complete'
  // - inputDelay
  // - processingDuration
  // - presentationDelay
  
  analytics.send({
    metric: 'INP',
    value: metric.value,
    target: metric.attribution.interactionTarget,
    type: metric.attribution.interactionType,
    phases: {
      inputDelay: metric.attribution.inputDelay,
      processing: metric.attribution.processingDuration,
      presentation: metric.attribution.presentationDelay
    }
  });
});
```

**Output example:**
```javascript
{
  value: 520,
  attribution: {
    interactionTarget: '<button class="submit-btn">',
    interactionType: 'pointer',
    inputDelay: 180,          // <- Bottleneck #1
    processingDuration: 280,  // <- Bottleneck #2
    presentationDelay: 60
  }
}
```

👉 **Biết được phase nào chiếm thời gian → biết fix gì**

---

#### Step 2: Phân tích từng phase

**Pattern 1: Input Delay cao (> 100ms)**
```
Symptoms:
- inputDelay > 100ms
- processingDuration OK
- presentationDelay OK

Root cause:
→ Main thread blocked khi user interact

Debug với Chrome DevTools Performance:
1. Record interaction
2. Tìm interaction marker
3. Xem main thread ngay TRƯỚC interaction
4. Long Task nào đang chạy?
```

**Fix strategies:**
```javascript
// ❌ Heavy work block main thread
function expensiveWork() {
  // 500ms synchronous work
}

// ✅ Break into chunks
async function expensiveWork() {
  for (let chunk of chunks) {
    await processChunk(chunk);
    // Yield to browser
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}

// ✅ Use scheduler API (React 18+)
import { startTransition } from 'react';

startTransition(() => {
  // Low priority work
  expensiveStateUpdate();
});
```

---

**Pattern 2: Processing Duration cao (> 100ms)**
```
Symptoms:
- inputDelay OK
- processingDuration > 100ms
- presentationDelay OK

Root cause:
→ Event handler quá nặng
→ React re-render quá nhiều components

Debug:
1. Chrome DevTools Performance: Xem JS execution trong event handler
2. React Profiler: Xem components nào render
```

**Fix strategies:**
```javascript
// ❌ Heavy logic in event handler
function handleClick() {
  const result = items.map(heavyTransform); // 200ms
  setData(result);
}

// ✅ Debounce user input
import { useDeferredValue } from 'react';

function SearchResults() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // Heavy filtering uses deferred value
  const results = useMemo(() => 
    items.filter(item => item.match(deferredQuery)),
    [deferredQuery]
  );
}

// ✅ Memoize expensive renders
const HeavyList = memo(({ items }) => {
  return items.map(item => <HeavyItem key={item.id} {...item} />);
});
```

---

**Pattern 3: Presentation Delay cao (> 50ms)**
```
Symptoms:
- inputDelay OK
- processingDuration OK
- presentationDelay > 50ms

Root cause:
→ Layout thrashing
→ Heavy paint/composite
→ CSS complexity

Debug:
Chrome DevTools Performance:
- Xem Rendering timeline
- Tìm Layout Shift
- Check Paint events
```

**Fix strategies:**
```javascript
// ❌ Force layout recalculation
function handleClick() {
  element.style.width = '100px';
  const height = element.offsetHeight; // 🚨 Force layout
  otherElement.style.height = height + 'px';
  const width = element.offsetWidth; // 🚨 Force layout again
}

// ✅ Batch reads and writes
function handleClick() {
  // Read phase
  const height = element.offsetHeight;
  const width = element.offsetWidth;
  
  // Write phase
  element.style.width = '100px';
  otherElement.style.height = height + 'px';
}

// ✅ Use CSS containment
.card {
  contain: layout style paint;
  /* Isolate rendering */
}

// ✅ Use transform instead of layout properties
// ❌
element.style.left = '100px'; // Trigger layout

// ✅
element.style.transform = 'translateX(100px)'; // Composite only
```

---

### 📊 Case Study: Dashboard với INP 800ms

**Initial state:**
```
INP: 800ms (Poor 🚨)

User action: Click filter button
Attribution:
- inputDelay: 250ms
- processingDuration: 450ms
- presentationDelay: 100ms
```

**Step 1: Fix Input Delay (250ms → 20ms)**

**Root cause:** Analytics script chạy Long Task

```javascript
// ❌ Before: Synchronous analytics
analytics.track('page_view', {
  // Heavy serialization: 200ms
  metadata: serializeEntireState()
});

// ✅ After: Defer analytics
queueMicrotask(() => {
  requestIdleCallback(() => {
    analytics.track('page_view', metadata);
  });
});
```

**Result:** inputDelay: 250ms → 20ms ✅

---

**Step 2: Fix Processing Duration (450ms → 80ms)**

**Root cause:** Re-render entire 1000-item table

```javascript
// ❌ Before
function handleFilterChange(newFilter) {
  setFilter(newFilter); // Trigger re-render of 1000 rows
}

function DataTable({ items, filter }) {
  // Filter in render (450ms)
  const filtered = items.filter(item => item.category === filter);
  
  return filtered.map(item => <Row key={item.id} {...item} />);
}

// ✅ After: Virtualization
import { FixedSizeList } from 'react-window';

function DataTable({ items, filter }) {
  const filtered = useMemo(
    () => items.filter(item => item.category === filter),
    [items, filter]
  );
  
  return (
    <FixedSizeList
      height={600}
      itemCount={filtered.length}
      itemSize={50}
    >
      {({ index, style }) => (
        <Row style={style} {...filtered[index]} />
      )}
    </FixedSizeList>
  );
}
```

**Result:** processingDuration: 450ms → 80ms ✅

---

**Step 3: Fix Presentation Delay (100ms → 30ms)**

**Root cause:** Layout thrashing trong row updates

```javascript
// ❌ Before
function Row({ data, isSelected }) {
  useEffect(() => {
    if (isSelected) {
      // Force layout
      const height = rowRef.current.offsetHeight;
      rowRef.current.style.marginTop = height + 'px';
    }
  }, [isSelected]);
}

// ✅ After: Use CSS
function Row({ data, isSelected }) {
  return (
    <div 
      className={isSelected ? 'row-selected' : 'row'}
      style={{ 
        transform: isSelected ? 'scale(1.02)' : 'scale(1)',
        transition: 'transform 0.2s'
      }}
    >
      {data.name}
    </div>
  );
}
```

**Result:** presentationDelay: 100ms → 30ms ✅

---

**Final result:**
```
Before:
- INP: 800ms (Poor 🚨)
- inputDelay: 250ms
- processingDuration: 450ms
- presentationDelay: 100ms

After:
- INP: 130ms (Good ✅)
- inputDelay: 20ms
- processingDuration: 80ms
- presentationDelay: 30ms

Impact:
- User satisfaction: +35%
- Task completion rate: +22%
```

---

### 💡 Advanced INP Optimization Techniques

#### 1. Event Handler Optimization

```javascript
// Pattern 1: Debounce frequent events
import { useDeferredValue } from 'react';

function SearchInput() {
  const [value, setValue] = useState('');
  const deferredValue = useDeferredValue(value);
  
  return (
    <>
      <input 
        value={value} 
        onChange={e => setValue(e.target.value)} // Instant UI update
      />
      <SearchResults query={deferredValue} /> {/* Deferred heavy work */}
    </>
  );
}

// Pattern 2: Passive event listeners
element.addEventListener('scroll', handler, { passive: true });
// → Browser can scroll immediately without waiting for handler
```

---

#### 2. Yield to Main Thread

```javascript
// ❌ Block main thread
function processLargeDataset(data) {
  return data.map(heavyTransform); // 500ms
}

// ✅ Yield periodically
async function processLargeDataset(data) {
  const results = [];
  const CHUNK_SIZE = 100;
  
  for (let i = 0; i < data.length; i += CHUNK_SIZE) {
    const chunk = data.slice(i, i + CHUNK_SIZE);
    results.push(...chunk.map(heavyTransform));
    
    // Yield to browser every 100 items
    await scheduler.yield(); // React Scheduler
    // or
    await new Promise(r => setTimeout(r, 0));
  }
  
  return results;
}
```

---

#### 3. requestIdleCallback for Low Priority Work

```javascript
function handleUserAction() {
  // High priority: Update UI
  updateUI();
  
  // Low priority: Analytics, logging
  requestIdleCallback(() => {
    sendAnalytics();
    updateCache();
  }, { timeout: 2000 });
}
```

---

### ⚠️ Common INP Anti-patterns

**Anti-pattern 1: Synchronous expensive work in event handler**
```javascript
// ❌
button.onclick = () => {
  const result = expensiveComputation(); // 300ms
  updateUI(result);
};

// ✅
button.onclick = async () => {
  updateUI({ loading: true }); // Instant feedback
  const result = await computeAsync();
  updateUI({ data: result, loading: false });
};
```

---

**Anti-pattern 2: Multiple state updates causing multiple renders**
```javascript
// ❌ Each setState triggers re-render
function handleSubmit() {
  setLoading(true);      // Render 1
  setError(null);        // Render 2
  setData(newData);      // Render 3
  setTimestamp(Date.now()); // Render 4
}

// ✅ Batch updates (React 18+)
function handleSubmit() {
  startTransition(() => {
    // All batched into 1 render
    setLoading(true);
    setError(null);
    setData(newData);
    setTimestamp(Date.now());
  });
}

// ✅ Or use reducer
function handleSubmit() {
  dispatch({
    type: 'SUBMIT',
    payload: { data: newData }
  });
  // 1 render only
}
```

---

**Anti-pattern 3: Layout thrashing**
```javascript
// ❌ Read-write-read-write
elements.forEach(el => {
  const height = el.offsetHeight; // Read (force layout)
  el.style.width = height + 'px'; // Write
  const width = el.offsetWidth;   // Read (force layout again)
  el.style.marginTop = width + 'px'; // Write
});

// ✅ Batch all reads, then all writes
const heights = elements.map(el => el.offsetHeight); // Read phase
const widths = elements.map(el => el.offsetWidth);

elements.forEach((el, i) => { // Write phase
  el.style.width = heights[i] + 'px';
  el.style.marginTop = widths[i] + 'px';
});
```

---

### 🎓 Tổng kết INP

#### ✅ Key Takeaways:

1. **INP thay thế FID** - Metric quan trọng nhất cho responsiveness (2024+)
2. **INP = 3 phases** - Input Delay + Processing + Presentation
3. **Debug theo phase** - Attribution API cho biết bottleneck ở đâu
4. **Optimize theo pattern** - Mỗi phase có strategies riêng
5. **Measure real users** - Lab metrics không đủ, cần RUM

#### 🎯 Quick Decision Framework:

```
INP > 500ms? 
├─ Check attribution
│  ├─ inputDelay cao → Optimize Long Tasks
│  ├─ processingDuration cao → Optimize event handlers / React renders
│  └─ presentationDelay cao → Fix layout thrashing / CSS
│
└─ Common fixes:
   1. Debounce/defer heavy work
   2. Break Long Tasks
   3. Virtualize long lists
   4. React.memo heavy components
   5. Batch layout reads/writes
```

---

## 1.16 PerformanceObserver API – Engine Đằng Sau Metrics

### 🎯 Tại sao phải học PerformanceObserver?

**Senior FE thường nghĩ:**
> "Có web-vitals library rồi, cần gì phải học PerformanceObserver?"

**Sai.**

**PerformanceObserver là:**
- Engine đằng sau tất cả performance metrics
- Cách để đo custom metrics (không có trong web-vitals)
- Cách để hiểu performance ở level sâu hơn

**Khi nào cần dùng trực tiếp:**
1. Đo custom business metrics
2. Track performance của specific features
3. Debug production issues (web-vitals không đủ chi tiết)
4. Implement custom monitoring solution

👉 **Không phụ thuộc library = hiểu bản chất**

---

### 📖 PerformanceObserver Basics

**Concept:**
Browser expose performance data qua `PerformanceEntry` objects.  
`PerformanceObserver` subscribe để nhận entries theo real-time.

**Syntax cơ bản:**
```javascript
const observer = new PerformanceObserver((list) => {
  // list.getEntries() = array of PerformanceEntry
  for (const entry of list.getEntries()) {
    console.log(entry.name, entry.duration);
  }
});

// Observe specific entry types
observer.observe({ entryTypes: ['navigation', 'resource'] });
```

**Performance Entry structure:**
```javascript
{
  name: 'https://example.com/api/data',
  entryType: 'resource',
  startTime: 1234.5,
  duration: 543.2,
  // ... type-specific properties
}
```

---

### 📊 Entry Types – Đo được gì?

#### 1. `navigation` – Page Load Timing

**Đo gì:**
- Document load performance
- DNS, TCP, request, response timing
- DOMContentLoaded, load events

```javascript
const observer = new PerformanceObserver((list) => {
  const navEntry = list.getEntries()[0];
  
  console.log('DNS lookup:', navEntry.domainLookupEnd - navEntry.domainLookupStart);
  console.log('TCP connection:', navEntry.connectEnd - navEntry.connectStart);
  console.log('TTFB:', navEntry.responseStart - navEntry.requestStart);
  console.log('DOM Content Loaded:', navEntry.domContentLoadedEventEnd);
  console.log('Load complete:', navEntry.loadEventEnd);
});

observer.observe({ entryTypes: ['navigation'] });
```

**Use case:**
- Debug slow server response
- Identify network issues
- Measure backend performance từ frontend

---

#### 2. `resource` – Resource Load Timing

**Đo gì:**
- Mỗi resource (JS, CSS, images, fonts, API calls)
- Timing breakdown (DNS, connect, download)

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // Chỉ track API calls
    if (entry.name.includes('/api/')) {
      console.log(`API: ${entry.name}`);
      console.log(`Duration: ${entry.duration}ms`);
      console.log(`Transfer size: ${entry.transferSize} bytes`);
      
      // Send to analytics
      analytics.send({
        type: 'api-timing',
        endpoint: entry.name,
        duration: entry.duration,
        size: entry.transferSize
      });
    }
  }
});

observer.observe({ entryTypes: ['resource'] });
```

**Use case:**
- Track API performance
- Monitor third-party scripts
- Detect slow resources

---

#### 3. `paint` – Paint Timing

**Đo gì:**
- First Paint (FP)
- First Contentful Paint (FCP)

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.startTime}ms`);
  }
});

observer.observe({ entryTypes: ['paint'] });

// Output:
// first-paint: 450ms
// first-contentful-paint: 680ms
```

**Use case:**
- Measure perceived load speed
- Optimize initial render
- Track painting regressions

---

#### 4. `largest-contentful-paint` – LCP

**Đo gì:**
- Largest image/text block render time
- Element that is LCP

```javascript
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1]; // LCP updates over time
  
  console.log('LCP:', lastEntry.renderTime || lastEntry.loadTime);
  console.log('LCP element:', lastEntry.element);
  console.log('LCP URL:', lastEntry.url); // If image
  
  // Send to analytics
  analytics.send({
    metric: 'LCP',
    value: lastEntry.renderTime || lastEntry.loadTime,
    element: lastEntry.element?.tagName,
    url: lastEntry.url
  });
});

observer.observe({ entryTypes: ['largest-contentful-paint'] });
```

**Important:** LCP updates multiple times. Always take the last entry.

---

#### 5. `layout-shift` – CLS

**Đo gì:**
- Individual layout shifts
- Elements that shifted
- Shift score

```javascript
let clsScore = 0;

const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // Ignore shifts caused by user input
    if (!entry.hadRecentInput) {
      clsScore += entry.value;
      
      console.log('Layout shift:', entry.value);
      console.log('Shifted elements:', entry.sources);
      
      // Track problem shifts (> 0.1)
      if (entry.value > 0.1) {
        analytics.send({
          metric: 'large-shift',
          value: entry.value,
          elements: entry.sources.map(s => ({
            node: s.node?.tagName,
            previousRect: s.previousRect,
            currentRect: s.currentRect
          }))
        });
      }
    }
  }
  
  console.log('Total CLS:', clsScore);
});

observer.observe({ entryTypes: ['layout-shift'] });
```

**Use case:**
- Debug CLS issues
- Identify which elements shift
- Correlate shifts với code changes

---

#### 6. `longtask` – Long Tasks (> 50ms)

**Đo gì:**
- Tasks block main thread > 50ms
- Attribution (script URL)

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('Long Task detected:', entry.duration, 'ms');
    console.log('Attribution:', entry.attribution);
    
    // Alert if task > 200ms
    if (entry.duration > 200) {
      analytics.send({
        metric: 'critical-long-task',
        duration: entry.duration,
        source: entry.attribution[0]?.containerSrc
      });
    }
  }
});

observer.observe({ entryTypes: ['longtask'] });
```

**Use case:**
- Detect blocking scripts
- Find performance bottlenecks
- Monitor third-party impact

---

#### 7. `measure` – Custom Marks & Measures

**Đo gì:**
- Custom performance marks
- Bạn tự define

```javascript
// Mark specific points
performance.mark('search-start');
// ... user code
performance.mark('search-end');

// Measure between marks
performance.measure('search-duration', 'search-start', 'search-end');

// Observe measurements
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.duration}ms`);
    
    analytics.send({
      metric: entry.name,
      duration: entry.duration
    });
  }
});

observer.observe({ entryTypes: ['measure'] });
```

**Use case:** (CỰC QUAN TRỌNG)
- Đo business-specific flows
- Track feature performance
- Custom metrics không có trong CWV

---

### 🎯 Real-World Patterns

#### Pattern 1: Track API Performance

```javascript
// Track tất cả API calls
const apiObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.initiatorType === 'fetch' || entry.initiatorType === 'xmlhttprequest') {
      const url = new URL(entry.name);
      
      analytics.send({
        type: 'api-performance',
        endpoint: url.pathname,
        duration: entry.duration,
        transferSize: entry.transferSize,
        // Categorize speed
        speed: entry.duration < 200 ? 'fast' : entry.duration < 1000 ? 'medium' : 'slow'
      });
      
      // Alert if > 3s
      if (entry.duration > 3000) {
        console.error(`Slow API: ${url.pathname} took ${entry.duration}ms`);
      }
    }
  }
});

apiObserver.observe({ entryTypes: ['resource'] });
```

---

#### Pattern 2: Measure Feature Time-to-Interactive

```javascript
// Example: Search feature
function setupSearchTracking() {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (entry.name === 'search-tti') {
        console.log('Search Time-to-Interactive:', entry.duration);
        
        analytics.send({
          feature: 'search',
          metric: 'time-to-interactive',
          duration: entry.duration
        });
      }
    }
  });
  
  observer.observe({ entryTypes: ['measure'] });
}

// Usage in app
function initializeSearch() {
  performance.mark('search-start');
  
  loadSearchIndex()
    .then(() => {
      performance.mark('search-ready');
      performance.measure('search-tti', 'search-start', 'search-ready');
    });
}
```

---

#### Pattern 3: Monitor Third-Party Scripts

```javascript
const thirdPartyObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    // Detect third-party by domain
    const url = new URL(entry.name);
    if (!url.hostname.includes('yourdomain.com')) {
      console.log(`Third-party: ${url.hostname}`);
      console.log(`Duration: ${entry.duration}ms`);
      console.log(`Transfer: ${entry.transferSize} bytes`);
      
      // Track heavy third-parties
      if (entry.duration > 1000 || entry.transferSize > 100000) {
        analytics.send({
          type: 'heavy-third-party',
          domain: url.hostname,
          duration: entry.duration,
          size: entry.transferSize
        });
      }
    }
  }
});

thirdPartyObserver.observe({ entryTypes: ['resource'] });
```

---

#### Pattern 4: Detect Performance Regressions

```javascript
// Baseline performance thresholds
const THRESHOLDS = {
  LCP: 2500,
  FCP: 1800,
  'api-duration': 500,
  'search-tti': 1000
};

function setupRegressionDetection() {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      const metric = entry.name || entry.entryType;
      const value = entry.duration || entry.renderTime || entry.startTime;
      
      if (THRESHOLDS[metric] && value > THRESHOLDS[metric]) {
        // Regression detected!
        console.warn(`⚠️ Performance regression: ${metric}`);
        console.warn(`Expected: < ${THRESHOLDS[metric]}ms, Got: ${value}ms`);
        
        analytics.send({
          type: 'performance-regression',
          metric,
          expected: THRESHOLDS[metric],
          actual: value,
          severity: value / THRESHOLDS[metric] // How bad
        });
      }
    }
  });
  
  observer.observe({ 
    entryTypes: ['largest-contentful-paint', 'paint', 'measure', 'resource'] 
  });
}
```

---

### 💡 Advanced Techniques

#### 1. Buffer Entries (Get Past Entries)

```javascript
// Observe + get entries that happened before observer was created
const observer = new PerformanceObserver((list) => {
  console.log('New entries:', list.getEntries());
});

observer.observe({ 
  entryTypes: ['resource'],
  buffered: true // ← Include past entries
});
```

**Use case:** Late initialization, single-page apps

---

#### 2. Disconnect Observer When Done

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === 'largest-contentful-paint') {
      console.log('LCP:', entry.renderTime);
      
      // Stop observing after LCP is stable (after 3s)
      setTimeout(() => {
        observer.disconnect();
        console.log('Stopped observing LCP');
      }, 3000);
    }
  }
});

observer.observe({ entryTypes: ['largest-contentful-paint'] });
```

**Use case:** Prevent memory leaks, stop tracking after key metrics

---

#### 3. Combine Multiple Entry Types

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    switch (entry.entryType) {
      case 'largest-contentful-paint':
        trackLCP(entry);
        break;
      case 'layout-shift':
        trackCLS(entry);
        break;
      case 'longtask':
        trackLongTask(entry);
        break;
      case 'measure':
        trackCustomMetric(entry);
        break;
    }
  }
});

observer.observe({ 
  entryTypes: ['largest-contentful-paint', 'layout-shift', 'longtask', 'measure'] 
});
```

---

### 📚 Case Study: E-commerce Checkout Performance

**Goal:** Track complete checkout flow performance

```javascript
// Setup tracking
function trackCheckoutPerformance() {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      // Custom metrics
      if (entry.entryType === 'measure') {
        const metrics = {
          'load-cart': { threshold: 500, priority: 'high' },
          'validate-form': { threshold: 200, priority: 'high' },
          'submit-order': { threshold: 2000, priority: 'critical' },
          'calculate-shipping': { threshold: 1000, priority: 'medium' }
        };
        
        const config = metrics[entry.name];
        if (config) {
          const isSlowvalue: entry.duration,
 = entry.duration > config.threshold;
          
          analytics.send({
            flow: 'checkout',
            step: entry.name,
            duration: entry.duration,
            threshold: config.threshold,
            isSlow,
            priority: config.priority
          });
          
          if (isSlow && config.priority === 'critical') {
            // Alert for critical slow steps
            alertOps({
              message: `Checkout ${entry.name} is slow: ${entry.duration}ms`,
              severity: 'high'
            });
          }
        }
      }
    }
  });
  
  observer.observe({ entryTypes: ['measure'], buffered: true });
}

// Usage in checkout flow
async function handleCheckout() {
  // Step 1: Load cart
  performance.mark('cart-start');
  await loadCart();
  performance.mark('cart-end');
  performance.measure('load-cart', 'cart-start', 'cart-end');
  
  // Step 2: Validate form
  performance.mark('validate-start');
  const isValid = await validateForm();
  performance.mark('validate-end');
  performance.measure('validate-form', 'validate-start', 'validate-end');
  
  // Step 3: Calculate shipping
  performance.mark('shipping-start');
  await calculateShipping();
  performance.mark('shipping-end');
  performance.measure('calculate-shipping', 'shipping-start', 'shipping-end');
  
  // Step 4: Submit order
  performance.mark('submit-start');
  await submitOrder();
  performance.mark('submit-end');
  performance.measure('submit-order', 'submit-start', 'submit-end');
}
```

**Result:** Phát hiện `calculate-shipping` thường > 2s → optimize → conversion +12%

---

### ⚠️ Best Practices & Gotchas

#### ✅ DO:

1. **Always handle entry types gracefully**
```javascript
try {
  observer.observe({ entryTypes: ['longtask'] });
} catch (e) {
  console.warn('Long Task API not supported');
  // Fallback logic
}
```

2. **Disconnect observers khi không cần**
```javascript
window.addEventListener('beforeunload', () => {
  observer.disconnect();
});
```

3. **Use buffered cho late observers**
```javascript
observer.observe({ entryTypes: ['paint'], buffered: true });
```

---

#### ❌ DON'T:

1. **Đừng observe quá nhiều entry types không cần**
```javascript
// ❌ Waste resources
observer.observe({ entryTypes: PerformanceObserver.supportedEntryTypes });

// ✅ Chỉ observe cần thiết
observer.observe({ entryTypes: ['measure', 'largest-contentful-paint'] });
```

2. **Đừng gửi mọi entry lên analytics**
```javascript
// ❌ Too much data
observer.observe({ entryTypes: ['resource'] });
// → 1000s of resources

// ✅ Filter before sending
if (entry.duration > 1000 || entry.transferSize > 100000) {
  analytics.send(entry);
}
```

3. **Đừng quên clear marks/measures**
```javascript
// Sau khi measure
performance.clearMarks('search-start');
performance.clearMeasures('search-duration');
```

---

### 🎓 Tổng kết PerformanceObserver

#### ✅ Key Takeaways:

1. **PerformanceObserver = Browser API** để observe performance entries real-time
2. **Entry Types** - navigation, resource, paint, LCP, CLS, longtask, measure
3. **Custom metrics** - Dùng marks + measures cho business logic
4. **Production monitoring** - Track real users, không chỉ lab
5. **Không phụ thuộc library** - Hiểu bản chất, flexible hơn

#### 📋 Reference: Supported Entry Types

| Entry Type | Đo gì | Use case |
|-----------|-------|----------|
| `navigation` | Page load timing | Debug slow load |
| `resource` | Resource timing | Track API, images, scripts |
| `paint` | FP, FCP | Perceived load speed |
| `largest-contentful-paint` | LCP | Main content render |
| `first-input` | FID (deprecated) | Use INP instead |
| `layout-shift` | CLS | Debug shifting elements |
| `longtask` | Tasks > 50ms | Find blocking code |
| `measure` | Custom metrics | Business-specific tracking |
| `element` | Element timing | Track specific elements |

#### 🔗 Resources:

- MDN: https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver
- W3C Spec: https://w3c.github.io/performance-timeline/
- Browser support: https://caniuse.com/mdn-api_performanceobserver

---



## 🎓 TỔ KẾT: Đo Performance Như Một Principal FE

### 📋 Tools Comparison Matrix

Để Senior FE **không lạc** giữa hàng chục tools, đây là decision matrix:

| Tool / Metric | Khi nào dùng | Dùng để làm gì | Không dùng khi |
|--------------|-------------|----------------|----------------|
| **Chrome DevTools Performance** | Debug specific issue | Tìm bottleneck (CPU/Network/Rendering) | Initial assessment, high-level metrics |
| **Lighthouse** | Quick audit, compare before/after | Tìm quick wins, verify fixes | Production data, complex apps |
| **Web Vitals (CrUX)** | Production monitoring | Understand real user experience | Debugging (không đủ chi tiết) |
| **React Profiler** | React re-render issues | Tìm wasted renders, component bottlenecks | Non-React issues, initial load |
| **Bundle Analyzer** | Bundle too large | Identify duplicate/unused code | Runtime performance issues |
| **PerformanceObserver** | Custom metrics, production tracking | Business-specific flows, RUM | Quick debugging (quá low-level) |
| **Network Tab** | Resource loading issues | Debug slow API, late resource discovery | JavaScript execution issues |

---

### 🔄 Complete Measurement Workflow

**Scenario: "App chậm" – Workflow từ report đến fix**

```
Step 1: Thu thập context
├─ User complaint hoặc analytics alert?
├─ Specific page / flow nào?
├─ Ảnh hưởng bao nhiêu users? (RUM data)
└─ Lab metrics có reproduce được không?

Step 2: High-level assessment
├─ Check CWV (LCP, INP, CLS) từ field data
├─ Run Lighthouse → quick wins?
├─ Check bundle size → có spike không?
└─ Xác định category: Load slow, Interaction slow, hay Visual instability?

Step 3: Deep dive (theo category)

📊 Nếu Load slow (LCP cao):
├─ Chrome DevTools Performance
│  ├─ Network waterfall → resource loading
│  ├─ Main thread → JS execution
│  └─ Rendering timeline → paint/layout
├─ Network Tab
│  ├─ Slow resources → LCP element?
│  ├─ Resource priority → correct?
│  └─ Late discovery → need preload?
└─ Bundle Analyzer
   └─ Initial bundle quá lớn → code splitting?

🖱️ Nếu Interaction slow (INP cao):
├─ PerformanceObserver + web-vitals
│  └─ Attribution → Input delay / Processing / Presentation?
├─ Chrome DevTools Performance
│  ├─ Long Tasks → break tasks
│  ├─ Event handler → excessive work?
│  └─ Layout thrashing → batch reads/writes?
└─ React Profiler (nếu React)
   └─ Wasted renders → memo / context split?

🎨 Nếu Visual instability (CLS cao):
├─ Chrome DevTools Performance
│  └─ Layout Shift events → which elements?
├─ PerformanceObserver (layout-shift)
│  └─ Shift sources → reserve space?
└─ Manual inspection
   └─ Missing width/height → ads / images?

Step 4: Implement fix

Step 5: Verify fix
├─ Lab: Lighthouse before/after
├─ Local: DevTools Performance
├─ Staging: Synthetic monitoring
└─ Production: RUM (wait 7 days for statistical significance)

Step 6: Prevent regression
├─ Performance budget in CI
├─ Bundle size checks
└─ Continuous monitoring (alerts)
```

---

### ✅ Checklist: Senior → Principal Level

**Senior FE thường:**
- ✅ Biết chạy Lighthouse
- ✅ Đọc được CWV metrics
- ✅ Dùng DevTools Performance tab
- ❌ Nhưng không biết khi nào dùng tool nào
- ❌ Không hiểu bản chất metrics
- ❌ Không track production data
- ❌ Không correlate metrics với business impact

**Principal FE phải:**
- ✅ Hiểu **TẠI SAO** đo từng metric (không chỉ HOW)
- ✅ Phân biệt được **Lab vs Field vs Business metrics**
- ✅ Chọn đúng tool cho từng scenario
- ✅ Implement **custom metrics** cho business logic
- ✅ Setup **continuous monitoring** (không chỉ one-time audit)
- ✅ **Correlate performance với business metrics** (conversion, bounce rate)
- ✅ **Prevent regressions** (performance budgets, CI checks)
- ✅ **Dạy người khác** (không chỉ tự biết)

---

### 🎯 Quick Reference: "Tôi nên xem gì trước?"

#### Scenario 1: User report "Page load chậm"
```
1. Lighthouse (5 phút) → LCP, FCP, TBT
2. Network Tab → TTFB, slow resources
3. DevTools Performance → Network waterfall + main thread
4. Bundle Analyzer → Initial bundle size
```

#### Scenario 2: User report "Click không phản hồi"
```
1. web-vitals + PerformanceObserver → INP attribution
2. DevTools Performance → Long Tasks around interaction
3. React Profiler (nếu React) → Wasted renders
```

#### Scenario 3: User report "Content nhảy lung tung"
```
1. DevTools Performance → Layout Shift events
2. PerformanceObserver (layout-shift) → Shift sources
3. Manual inspect → Missing dimensions
```

#### Scenario 4: "Metric regression sau deploy"
```
1. RUM dashboard → Which metric spiked?
2. Git diff → Code changes in deploy
3. Bundle Analyzer compare → Bundle size increase?
4. Synthetic monitoring → Reproduce in controlled environment
```

---

### 📚 Resources & Next Steps

#### Đọc tiếp:
- **MDN Performance APIs**: https://developer.mozilla.org/en-US/docs/Web/API/Performance_API
- **Web Vitals Guide**: https://web.dev/vitals/
- **Chrome DevTools Docs**: https://developer.chrome.com/docs/devtools/performance/

#### Hands-on practice:
1. **Setup RUM** cho project hiện tại (web-vitals + analytics)
2. **Implement custom metrics** cho 1 critical flow (checkout, search, etc)
3. **Performance budget** trong CI pipeline
4. **Weekly review** performance metrics (track trends)

#### Advanced topics (separate documents):
- **Network Optimization** - Bundle splitting, lazy loading, caching
- **Runtime Performance** - Debouncing, web workers, layout optimization  
- **Memory Management** - Leak detection, cleanup patterns

---

### 💬 Lời Kết

**Performance measurement không phải để:**
- Khoe điểm số
- So sánh team
- Làm đẹp dashboard

**Performance measurement để:**
- **Ra quyết định đúng** (optimize cái gì trước?)
- **Phát hiện regression sớm** (trước khi user complain)
- **Gắn với business impact** (ROI của optimization)
- **Improve UX liên tục** (data-driven, không phải gut feeling)

👉 **Metric là means, không phải end. End luôn là better user experience.**

---

**DOCUMENT COMPLETE ✅**

*Version: 1.0*  
*Last updated: 2026-01-02*  
*Focus: MEASUREMENT (not optimization)*  
*Next: Network Optimization, Runtime Performance, Memory Management*

