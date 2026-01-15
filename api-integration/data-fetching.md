# 5. Data Fetching Strategies

> **Data fetching không phải là gọi API**
> mà là **quyết định: dữ liệu được lấy ở đâu, khi nào, và ai chịu trách nhiệm**

Sai strategy → app chậm, khó scale, khó maintain
Đúng strategy → performance + DX + SEO cùng thắng

---

## 5.1 Client-side Fetching (useEffect)

### Bản chất

> **Fetch sau khi UI đã render**

Flow:

```
Render UI → JS chạy → gọi API → setState → re-render
```

---

### Khi nào nên dùng

* Data **user-specific**
* Không cần SEO
* Interaction-driven (click, filter, pagination)
* Auth-required data

---

### Ưu điểm

* Đơn giản
* Dễ debug
* Không phụ thuộc server render

---

### Hạn chế bản chất

* Loading state bắt buộc
* SEO kém
* Waterfall request dễ xảy ra

---

### Anti-patterns

❌ Fetch trong nhiều useEffect chồng chéo
❌ Fetch theo component tree sâu
❌ UI phụ thuộc thứ tự fetch

> useEffect không sai — **lạm dụng mới sai**

---

## 5.2 Server-side Rendering (SSR – Next.js)

### Bản chất

> **Fetch data trước khi HTML được gửi về browser**

Flow:

```
Server fetch data → render HTML → client hydrate
```

---

### Khi nào nên dùng

* SEO quan trọng
* First Contentful Paint quan trọng
* Page-level data

---

### Ưu điểm

* SEO tốt
* UX nhanh hơn lần đầu
* Data sẵn khi render

---

### Hạn chế bản chất

* Server load cao hơn
* TTFB có thể tăng
* Phức tạp hơn client-only

---

### Sai lầm phổ biến

❌ Fetch quá nhiều data ở SSR
❌ SSR cho data cá nhân nặng
❌ Không cache SSR result

---

## 5.3 Static Generation (SSG)

### Bản chất

> **Fetch data tại build time**

HTML được build sẵn, serve như file tĩnh.

---

### Khi nào nên dùng

* Content ít thay đổi
* Public data
* Marketing, blog, docs

---

### Ưu điểm

* Rất nhanh
* Cache CDN cực tốt
* Server cost thấp

---

### Hạn chế

* Data stale
* Không phù hợp user-specific
* Build time tăng nếu scale lớn

---

### Anti-patterns

❌ Dùng SSG cho data dynamic
❌ Rebuild toàn site vì thay đổi nhỏ

---

## 5.4 Incremental Static Regeneration (ISR)

### Bản chất

> **Static + Update theo thời gian**

* Serve HTML cũ
* Regenerate background
* Lần sau user nhận bản mới

---

### Khi nào nên dùng

* Data thay đổi chậm
* Chấp nhận stale data ngắn
* Page nhiều nhưng không rebuild toàn bộ

---

### Trade-off quan trọng

* Consistency vs performance
* User A và B có thể thấy data khác nhau

---

### Sai lầm phổ biến

❌ Assume data luôn fresh
❌ Không communicate stale behavior cho product

---

## 5.5 React Server Components (RSC)

### Bản chất (rất quan trọng)

> **Component chạy trên server, không ship JS xuống client**

RSC không phải SSR.

---

### RSC giải quyết gì?

* Giảm bundle size
* Fetch data gần source
* Không expose secret

---

### Khi nào nên dùng

* Data-heavy UI
* Read-only content
* Layout / shell

---

### Khi KHÔNG nên dùng

* Interactive logic
* Browser-only APIs
* State phức tạp phía client

---

### Sai lầm phổ biến

❌ Dùng RSC cho mọi component
❌ Trộn server/client boundary không rõ

> RSC = powerful nhưng **rất unforgiving**

---

## 5.6 So sánh nhanh (mental model)

| Strategy | Fetch khi nào | SEO | Complexity |
| -------- | ------------- | --- | ---------- |
| CSR      | After render  | ❌   | Thấp       |
| SSR      | Request time  | ✅   | Trung      |
| SSG      | Build time    | ✅   | Thấp       |
| ISR      | Hybrid        | ✅   | Trung      |
| RSC      | Server render | ✅   | Cao        |

---

## 5.7 Principal Mental Checklist

Trước khi chọn strategy:

* Data này public hay private?
* SEO có quan trọng không?
* Tần suất update?
* User có chấp nhận stale data không?
* Server cost chấp nhận được không?

---

## Kết luận ngắn gọn

> **Không có strategy “tốt nhất”**
> chỉ có strategy **phù hợp nhất với data & UX**

## 5.8 Prefetching & Preloading

### Bản chất

> **Fetch trước khi user cần = faster UX**

---

### Strategies

**1. Link hover prefetch**
User hover link → prefetch next page

**2. Viewport prefetch**
Link xuất hiện trong viewport → prefetch

**3. Predictive prefetch**
Dựa trên user behavior pattern

---

### Trade-offs

* Bandwidth cost
* Cache pollution
* May fetch unused data

---

### Best practices

* Only prefetch likely navigation
* Use low priority
* Respect connection quality (slow 3G → skip)

---

## 5.9 Parallel vs Sequential Fetching

### Bản chất

> **Waterfall requests = #1 performance killer**

---

### Anti-pattern
```
fetch user → wait
fetch posts (needs user.id) → wait
fetch comments (needs posts.ids)
```

Total: 3 RTT

---

### Better
```
Promise.all([
  fetch user,
  fetch posts,
  fetch comments
])
```

Total: 1 RTT

---

### When sequential is needed

* Auth token → subsequent requests
* Pagination (need prev page result)
* Dependent queries

---

## 5.10 Data Fetching Anti-patterns Summary

❌ Fetch trong useEffect cascade
❌ Fetch mỗi component một
❌ Không handle loading states
❌ Fetch duplicate data
❌ Không prefetch obvious next steps
❌ Block render waiting for all data
❌ Fetch data không dùng

---
