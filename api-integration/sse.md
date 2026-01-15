# Server-Sent Events (SSE) – One-way Real-time

> **SSE không phải là WebSocket lite**
> mà là **HTTP-based server push, đơn giản và đủ dùng**

Nếu WebSocket là "điện thoại hai chiều"
thì SSE là:

> **"Radio broadcast – server nói, client nghe"**

---

## 4.1 When to use SSE vs WebSocket

### Bản chất

> **SSE = server push qua HTTP**
> Không cần protocol riêng, không cần hai chiều

---

### Use cases phù hợp

* Live notifications
* Progress updates (file upload, processing)
* Live feeds (news, social media)
* Server logs streaming
* Stock price updates (one-way)

👉 Rule of thumb:

> Nếu chỉ cần **server → client** → SSE
> Nếu cần **client ↔ server** → WebSocket

---

### So sánh nhanh

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| Direction | Server → Client | Bidirectional |
| Protocol | HTTP | WS protocol |
| Reconnect | Auto | Manual |
| Browser support | Good (IE không có) | Excellent |
| Complexity | Low | High |
| Firewall friendly | Yes | Sometimes blocked |

---

## 4.2 Connection Lifecycle

### Bản chất

> **SSE là long-lived HTTP connection**

Browser tự động:
* Reconnect khi disconnect
* Theo chuẩn 3 giây retry
* Gửi last-event-id

---

### EventSource API
```javascript
const eventSource = new EventSource('/api/events');

eventSource.onmessage = (event) => {
  // Handle message
};

eventSource.onerror = (error) => {
  // Auto reconnect, but you can close
  if (criticalError) {
    eventSource.close();
  }
};
```

---

### Nguyên tắc

* Browser tự reconnect → đừng implement lại
* Server phải support keep-alive
* Client phải close khi không dùng

---

## 4.3 Message Format

### Server phải trả về đúng format
```
data: {"type": "notification", "message": "Hello"}

data: Line 1
data: Line 2

event: custom-event
data: {"status": "complete"}
id: 12345
```

---

### Các field quan trọng

* `data:` – message content
* `event:` – custom event type
* `id:` – để resume nếu reconnect
* `retry:` – custom reconnect interval

---

### Anti-patterns

❌ Không set Content-Type: text/event-stream
❌ Buffer response (phải flush ngay)
❌ Không gửi comment để keep-alive

---

## 4.4 Authentication

### Bản chất

> **SSE dùng HTTP → auth qua headers hoặc query**

---

### Strategies

**Option 1: Token in URL**
```javascript
new EventSource('/events?token=abc123')
```

⚠️ Token lộ trong URL, log, history

**Option 2: Cookie-based**
```javascript
// Server set HttpOnly cookie
// Browser tự động gửi cookie
```

✅ An toàn hơn

**Option 3: Custom headers (không được)**
EventSource API không support custom headers

---

### Best practice

* Dùng short-lived token
* Validate token mỗi message
* Close connection nếu token expire

---

## 4.5 Scaling Considerations

### Bản chất

> **SSE = long connection = stateful**

Scale khó hơn REST.

---

### Challenges

* 10k users = 10k open connections
* Load balancer phải support sticky session
* Server memory tăng
* Không share state giữa server instances

---

### Solutions

* Dùng Redis Pub/Sub
* Message queue (RabbitMQ, Kafka)
* Dedicated SSE server
* Rate limit connections per user

---

### Anti-patterns

❌ Mỗi server giữ state riêng
❌ Không limit concurrent connections
❌ Không timeout idle connections

---

## 4.6 Error Handling & Fallback

### Bản chất

> **SSE có thể bị block bởi proxy/firewall**

---

### Fallback strategy
```
1. Try SSE
2. If fail after 3 attempts → Polling
3. Thông báo user về degraded experience
```

---

### Detection
```javascript
let failCount = 0;

eventSource.onerror = () => {
  failCount++;
  if (failCount > 3) {
    eventSource.close();
    fallbackToPolling();
  }
};
```

---

## 4.7 SSE Mental Checklist

Trước khi dùng SSE:

* Có cần two-way communication không?
* Server có support streaming không?
* Scale bao nhiêu concurrent users?
* Fallback strategy là gì?
* Auth mechanism nào phù hợp?

---

## So sánh cuối: SSE vs WebSocket vs Polling

| Criteria | SSE | WebSocket | Polling |
|----------|-----|-----------|---------|
| Latency | Low | Lowest | High |
| Overhead | Low | Medium | High |
| Complexity | Low | High | Low |
| Browser support | Good | Excellent | Universal |
| Firewall friendly | Yes | Sometimes | Yes |
| Bidirectional | No | Yes | Yes (inefficient) |

---

## Kết luận

> **SSE là sweet spot cho server push**
> Đơn giản hơn WebSocket, tốt hơn Polling
> 
> Default choice for one-way real-time