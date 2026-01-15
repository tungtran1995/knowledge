# 6. Caching – Make the System Fast *Without Lying*

> **Caching không phải là lưu dữ liệu lại**
> mà là **chấp nhận trade-off giữa freshness và performance**

Sai cache → bug rất khó debug
Đúng cache → hệ thống nhanh, rẻ, ổn định

---

## 6.1 Browser Cache (Cache-Control headers)

### Bản chất

> **Browser cache là cache đầu tiên và rẻ nhất**

Được điều khiển bởi **HTTP headers**, không phải JS.

---

### Khi nào nên dùng

* Static assets (JS, CSS, images)
* Public content
* Versioned resources

---

### Nguyên tắc cốt lõi

* **Immutable + versioned**
* Cache lâu, invalidate bằng version

---

### Sai lầm phổ biến

❌ Cache HTML dynamic
❌ Không set Cache-Control
❌ Dùng cache để “fix performance” mù quáng

> Browser cache mạnh nhưng **rất ngu** — nó làm đúng header bạn nói.

---

## 6.2 React Query Cache

### Bản chất

> **React Query cache = cache trạng thái async**, không phải cache HTTP

Nó trả lời:

> *“Dữ liệu này còn dùng được không?”*

---

### Khi nào nên dùng

* Client-side fetching
* User-specific data
* Data cần sync giữa nhiều component

---

### Khái niệm quan trọng

* Stale vs fresh
* Background refetch
* Cache invalidation theo key

---

### Sai lầm phổ biến

❌ Cache vô hạn
❌ Query key không ổn định
❌ Manual refetch khắp nơi

> React Query tốt khi bạn **tin vào nó**, không override nó liên tục.

---

## 6.3 Service Worker Cache

### Bản chất

> **Service Worker là proxy chạy trong browser**

Nó quyết định:

* Request nào được cache
* Serve từ đâu
* Khi nào fallback network

---

### Khi nào nên dùng

* Offline support
* PWA
* Static assets
* Predictable request pattern

---

### Trade-off

* Rất mạnh
* Rất dễ gây bug “ma”

---

### Sai lầm phổ biến

❌ Cache API dynamic
❌ Không clear cache khi deploy
❌ Không có versioning strategy

> SW bug = “user thấy lỗi mà bạn không reproduce được”

---

## 6.4 localStorage / sessionStorage

### Bản chất

> **Storage ≠ Cache**

* Không có expiration
* Không sync
* Không invalidation

---

### Khi nào nên dùng

* UI preference
* Feature flag client-side
* Draft tạm thời

---

### Khi KHÔNG nên dùng

* Auth token
* Source of truth
* Sensitive data

---

### Sai lầm phổ biến

❌ Dùng localStorage như DB
❌ Store security-critical data
❌ Không handle stale data

---

## 6.5 Cache Invalidation – The Hardest Problem

### Bản chất

> **Cache sai không fail rõ ràng — nó fail âm thầm**

Câu hỏi cần trả lời:

* Khi nào cache hết hạn?
* Ai invalidate?
* User có chấp nhận stale không?

---

### Nguyên tắc thực tế

* Cache gần nơi consume
* Invalidate theo event
* Prefer stale-while-revalidate

---

## 6.6 Caching Mental Model (Principal)

Hãy phân tầng cache:

1. Browser cache (free)
2. CDN / edge
3. Client state cache
4. App logic fallback

> Cache càng xa source → càng khó đúng

---

## 6.7 Caching Checklist

Trước khi cache:

* Data này thay đổi bao lâu?
* Nếu stale thì sao?
* Ai chịu trách nhiệm invalidate?
* Có dễ debug không?

---

## 6.8 HTTP Caching Deep Dive

### Headers chi tiết

**Cache-Control directives:**
```
public – CDN có thể cache
private – chỉ browser cache
no-cache – revalidate mỗi lần
no-store – không cache gì cả
max-age=3600 – cache 1 giờ
s-maxage=7200 – CDN cache 2 giờ
immutable – never revalidate
stale-while-revalidate=60 – serve stale, update background
```

---

### ETag vs Last-Modified

**ETag:**
* Hash của content
* Chính xác
* Server phải generate

**Last-Modified:**
* Timestamp
* Coarse-grained
* Đơn giản hơn

---

### Conditional requests
```
If-None-Match: "etag123"
→ 304 Not Modified (cache vẫn dùng được)

If-Modified-Since: Mon, 01 Jan 2024
→ 304 Not Modified
```

---

## 6.9 CDN Caching Strategy

### Bản chất

> **CDN là cache layer gần user nhất**

---

### Best practices

* Static assets: max-age=31536000, immutable
* API responses: s-maxage=60, stale-while-revalidate
* HTML: no-cache (revalidate luôn)
* Versioned URLs: cache forever

---

### Cache purging

* Purge by URL
* Purge by tag
* Purge all (avoid!)

---

## 6.10 React Query Advanced Patterns

### Optimistic Updates
```
Update UI trước
→ Call API
→ Nếu fail: rollback
```

---

### Dependent Queries
```
Query B chỉ chạy khi Query A xong
enabled: !!userId
```

---

### Infinite Queries

Pagination với auto-fetch on scroll

---

### Query Cancellation

Cancel ongoing request khi component unmount

---

## Kết luận cuối cho toàn bộ API & Data series

> **Performance tốt không đến từ fetch nhanh**
> mà từ **fetch ít, đúng lúc, và cache có kỷ luật**


