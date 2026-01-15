# 2. GraphQL – Flexible API, Sharp Tool

> **GraphQL không phải REST v2**
> mà là **một query language + execution model**.

Nếu REST trả lời:

> *“Endpoint này làm gì?”*

thì GraphQL trả lời:

> **“Client cần data gì, theo shape nào?”**

---

## 2.1 Queries vs Mutations

### Bản chất

> **Queries = đọc dữ liệu**
> **Mutations = thay đổi trạng thái**

Nghe giống GET vs POST, nhưng **khác bản chất**.

---

### Query

* Không side effect (theo design)
* Có thể cache
* Có thể gọi nhiều lần

Ví dụ tư duy:

> “Tôi muốn **shape data này**, không phải endpoint kia”

---

### Mutation

* Có side effect
* Có thể trả về data mới
* Thường trigger refetch / cache update

⚠️ Nguyên tắc:

> **Mutation phải mô tả hành động business**, không chỉ CRUD

Anti-pattern:

* `updateUserName`
* `setFieldX`

Tốt hơn:

* `changeUserEmail`
* `approveInvoice`

---

## 2.2 Fragments

### Bản chất

> **Fragment = contract tái sử dụng shape dữ liệu**

Không phải để “đỡ gõ”, mà để:

* Giữ consistency
* Tránh data drift giữa các screen

---

### Khi nào nên dùng fragment?

* Component-level data
* Nhiều query dùng chung shape

Anti-pattern:

* Fragment quá lớn
* Fragment chứa data không dùng

> Fragment nên map 1–1 với UI responsibility

---

## 2.3 Variables

### Bản chất

> **Variables = tách query shape khỏi giá trị runtime**

Giúp:

* Cache hiệu quả
* Tránh query injection
* Query dễ đọc hơn

---

### Anti-patterns

❌ Hardcode value trong query
❌ Generate query string động

> Query càng ổn định → cache càng tốt

---

## 2.4 Error Handling trong GraphQL

### Bản chất (rất hay bị hiểu sai)

> **GraphQL cho phép “partial success”**

Response có thể:

* Có `data`
* Có `errors`

Điều này **không phải bug**, mà là design.

---

### Các loại lỗi

* **GraphQL errors**: schema, validation
* **Business errors**: domain rule
* **Network errors**: transport

👉 Client **phải phân biệt**, không treat mọi error giống nhau.

---

### Anti-patterns

❌ Dùng HTTP 200 cho mọi lỗi mà không encode semantic
❌ Parse message string
❌ Không log error path / field

---

## 2.5 Caching Strategies (Apollo, urql)

### Bản chất

> **GraphQL cache theo entity, không theo endpoint**

Khác REST:

* REST cache = URL-based
* GraphQL cache = normalized entities

---

### Cache hoạt động tốt khi:

* Schema có `id` ổn định
* Query shape nhất quán
* Mutation trả về entity mới

---

### Common strategies

* **Normalized cache** (Apollo, urql graphcache)
* **Refetch queries**
* **Manual cache update**

Trade-off:

* Cache mạnh → complexity tăng

---

### Anti-patterns

❌ Cache everything
❌ Cache mutation result mù quáng
❌ Schema thiếu ID

---

## 2.6 When to use GraphQL vs REST

### GraphQL phù hợp khi:

* Client cần **data shape linh hoạt**
* Nhiều platform (web, mobile)
* UI phức tạp, nhiều view trên cùng data
* Overfetch / underfetch là vấn đề

---

### REST phù hợp khi:

* API đơn giản
* Action-based (export, import, webhook)
* Streaming / file upload
* Cache CDN mạnh

---

### Nguyên tắc thực tế (Principal)

> **GraphQL cho UI-driven data**
> **REST cho system-to-system & action-based APIs**

Không cần chọn 1 trong 2 cho toàn bộ hệ thống.

---

## 2.7 GraphQL Mental Checklist

Trước khi chọn GraphQL:

* Client có thực sự cần shape linh hoạt không?
* Team có đủ discipline cho schema & cache không?
* Debug & monitoring có sẵn chưa?
* Security (depth, complexity) đã nghĩ tới chưa?

---

## Kết nối với Security & Auth

* Auth → context
* AuthZ → field / resolver level
* Query depth → DoS risk
* Error path → tránh leak info

---

## 2.8 GraphQL Subscriptions

### Bản chất

> **Subscriptions = WebSocket-based real-time**

---

### Use cases

* Chat messages
* Live notifications
* Real-time collaboration
* Live sports scores

---

### Implementation

Thường dùng WebSocket transport
Client subscribe → server push updates

---

### Best practices

* Subscription resolver phải lightweight
* Rate limit subscriptions per user
* Auto-unsubscribe khi client disconnect
* Fallback to polling nếu WS không available

---

## 2.9 Schema Design Principles

### Bản chất

> **Schema là contract, design sai = pain lâu dài**

---

### Best practices

* Nullable vs non-nullable: cẩn thận
* ID type cho mọi entity
* Enums cho fixed values
* Connection pattern cho lists
* Error trong schema, không throw

---

### Anti-patterns

❌ Schema quá nested
❌ Circular references không kiểm soát
❌ Nullable everywhere
❌ String cho mọi thứ

---

## 2.10 GraphQL Performance

### N+1 Problem
```
Query 10 users → 1 query
Each user's posts → 10 queries
Total: 11 queries
```

---

### Solution: DataLoader

Batch + cache trong single request lifecycle

---

### Query Complexity Analysis

* Prevent expensive queries
* Limit query depth
* Calculate cost based on fields

---

## Kết luận ngắn gọn

> **GraphQL rất mạnh, nhưng không forgiving**
> Dùng đúng → productivity cao
> Dùng sai → complexity bùng nổ

