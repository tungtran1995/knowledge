# 1. REST APIs – Foundation

> **REST API không phải là HTTP + JSON**
> mà là **contract** giữa client và server.

Nếu Auth / Security trả lời *“ai được phép”*
thì REST API trả lời:

> **“Chúng ta nói chuyện với nhau như thế nào để không hiểu sai?”**

---

## 1.1 HTTP Methods (Intent, không phải hành động kỹ thuật)

### Bản chất

> **HTTP method thể hiện ý định (intent)** của request,
> không phải cách implement ở backend.

| Method | Ý nghĩa           | Ghi chú           |
| ------ | ----------------- | ----------------- |
| GET    | Lấy dữ liệu       | Không side effect |
| POST   | Tạo hành động mới | Không idempotent  |
| PUT    | Thay thế toàn bộ  | Idempotent        |
| PATCH  | Update một phần   | Thường dùng nhất  |
| DELETE | Xoá               | Có side effect    |

---

### Nguyên tắc quan trọng

* **GET phải an toàn (safe)**
* **PUT / DELETE phải idempotent**

> Nếu gọi 2 lần cho kết quả khác → API đang sai contract.

---

### Anti-patterns phổ biến

❌ GET để mutate data
❌ POST cho mọi thứ
❌ PATCH nhưng thực chất replace toàn bộ resource

---

## 1.2 Status Codes (Ngôn ngữ phản hồi của API)

### Bản chất

> **Status code = tín hiệu điều khiển flow**,
> không phải để “cho đẹp”.

---

### Nhóm quan trọng cần nhớ

#### 2xx – Thành công

* `200 OK`
* `201 Created`
* `204 No Content`

#### 4xx – Lỗi client

* `400 Bad Request` – input sai
* `401 Unauthorized` – chưa login
* `403 Forbidden` – không có quyền
* `404 Not Found` – không tồn tại / không cho biết tồn tại
* `409 Conflict` – trạng thái xung đột
* `422 Unprocessable Entity` – validate fail

#### 5xx – Lỗi server

* `500 Internal Error`
* `503 Service Unavailable`

---

### Nguyên tắc

* **Đừng dùng 200 cho mọi thứ**
* **401 ≠ 403**
* 404 đôi khi là **security decision**

---

## 1.3 Headers (Metadata quan trọng, hay bị xem nhẹ)

### Headers là gì?

> **Headers = metadata điều khiển hành vi**,
> không phải data business.

---

### Các header quan trọng

#### Authorization

* Chứa credential (token)
* Không log
* Không expose

#### Content-Type

* Server cần biết cách parse body
* Client cần biết cách đọc response

#### CORS headers

* **Không phải security**
* Chỉ là browser rule

---

### Anti-patterns

❌ Nhét token vào body
❌ Log Authorization header
❌ Wildcard CORS với credentials

---

## 1.4 Error Handling Strategies

### Bản chất

> **Error là một phần của API contract**

Client **phải hiểu được**:

* Có lỗi hay không
* Lỗi loại gì
* Có retry được không

---

### Error response nên có gì?

Concept-level:

```
- status code đúng
- error code ổn định
- message cho developer
```

👉 Đừng để client parse string message.

---

### Sai lầm phổ biến

❌ Trả cùng một error cho mọi case
❌ Leak stacktrace
❌ Message thay đổi theo ngôn ngữ

---

## 1.5 Retry Logic (At-least-once là mặc định)

### Bản chất

> **Mọi network request đều có thể fail**
> → retry là chuyện bình thường.

---

### Khi nào nên retry?

* Network error
* Timeout
* 5xx

❌ Không retry:

* 4xx
* Validation error

---

### Idempotency rất quan trọng

> Nếu retry gây side effect → API design có vấn đề.

Giải pháp:

* Idempotent methods
* Idempotency key (POST)

---

## 1.6 Timeout Handling

### Bản chất

> **Timeout = hệ thống tự bảo vệ mình**

Không có timeout:

* Request treo
* Resource bị giữ
* Cascade failure

---

### Nguyên tắc

* Client luôn có timeout
* Backend không assume client chờ mãi
* Không set timeout = ∞

---

### Anti-patterns

❌ “Đợi backend trả về”
❌ Retry vô hạn
❌ Block UI vì API chậm

---

## 1.7 REST API Mental Checklist

Trước khi thiết kế / review API:

* Method có đúng intent không?
* Status code có nói đúng trạng thái không?
* Lỗi này client xử lý thế nào?
* Retry có gây duplicate không?
* Timeout fail có graceful không?

---

## Kết nối với security

* Auth → gắn vào Authorization header
* Status code → tránh leak thông tin
* Retry → tránh duplicate action
* Timeout → tránh DoS gián tiếp

---

## Kết luận ngắn gọn

> **REST API tốt không phải là API “chạy được”**
> mà là API **không gây hiểu lầm cho client**.

---

## 1.8 API Versioning

### Bản chất

> **API sẽ thay đổi, versioning là cách communicate changes**

---

### Strategies

**1. URL versioning**
`/v1/users`, `/v2/users`

✅ Clear, explicit
❌ Duplicate endpoints

**2. Header versioning**
`Accept: application/vnd.api.v2+json`

✅ Clean URLs
❌ Harder to test manually

**3. Query param**
`/users?version=2`

❌ Don't use in production

---

### Best practices

* Support n và n-1 version
* Deprecation notice trước 6-12 tháng
* Document breaking changes rõ ràng
* Sunset header cho deprecated APIs

---

## 1.9 Rate Limiting

### Bản chất

> **Rate limiting protect cả client lẫn server**

---

### Headers cần return
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

---

### Client handling

* Respect 429 status
* Read Retry-After header
* Implement exponential backoff với jitter
* Queue requests nếu rate limit sắp hit

---

### Anti-patterns

❌ Retry immediately khi 429
❌ Không track rate limit locally
❌ Infinite retry loop

---

## 1.10 Request Deduplication

### Bản chất

> **Duplicate requests = waste + bugs**

React Suspense, concurrent rendering → vấn đề này trầm trọng

---

### Problem
```
Component A: fetch /user/me
Component B: fetch /user/me (duplicate!)
```

---

### Solution

* Request deduplication library
* Cache ongoing requests
* Return same promise cho identical requests

---

### Best practice

* Deduplicate trong fetch wrapper
* TTL ngắn (~100ms)
* Only for GET requests

---