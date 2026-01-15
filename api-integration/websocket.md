Ok 👍 mình sẽ **đi từng phần một, không gom**, giữ đúng format như các phần trước.
# 3. WebSocket – Real-time Communication

> **WebSocket không phải là “API nhanh hơn”**
> mà là **một connection sống, hai chiều, kéo dài**.

Nếu REST / GraphQL là:

> *“Hỏi – trả lời”*

thì WebSocket là:

> **“Giữ đường dây mở và nói chuyện liên tục”**

---

## 3.1 When to use WebSocket (Real-time requirements)

### Bản chất

> **WebSocket dùng khi server cần chủ động push data cho client, với độ trễ rất thấp**

Không phải “cái gì realtime cũng cần WebSocket”.

---

### Use cases phù hợp

* Chat, messaging
* Live collaboration (Google Docs)
* Realtime dashboard (stock, metrics)
* Online game
* Presence / typing indicator

👉 Điểm chung:

* **Latency thấp**
* **Server push**
* **Event liên tục**

---

### Khi KHÔNG nên dùng WebSocket

* Data update hiếm
* Request/response đơn giản
* SEO quan trọng
* Hệ thống chưa sẵn sàng scale connection

> WebSocket có cost **cao hơn REST rất nhiều**.

---

## 3.2 Connection Management

### Bản chất

> **WebSocket không phải request — nó là một resource sống**

Mỗi connection:

* Tốn memory
* Tốn CPU
* Tốn slot trên server

---

### Những thứ phải quản lý

* Open / close lifecycle
* Auth khi connect
* Cleanup khi disconnect
* Heartbeat / ping-pong

---

### Sai lầm phổ biến

❌ Mở connection nhưng không close
❌ Không detect connection chết
❌ Không giới hạn số connection

> WebSocket leak = memory leak cấp hệ thống

---

### Nguyên tắc

* Connection phải có lifecycle rõ ràng
* Auth **khi handshake**
* Server phải chủ động close connection xấu

---

## 3.3 Reconnection Strategies

### Bản chất

> **Connection sẽ fail — chắc chắn**

* Network chập chờn
* Tab background
* Mobile sleep
* Server restart

---

### Reconnect đúng cách

* Exponential backoff
* Giới hạn retry
* Re-auth nếu cần
* Resume state nếu có thể

---

### Sai lầm phổ biến

❌ Reconnect liên tục không delay
❌ Infinite reconnect
❌ Reconnect mà không sync lại state

> Reconnect sai → tự tạo DoS cho backend

---

### Câu hỏi cần trả lời khi reconnect

* Missed message xử lý thế nào?
* Có cần fetch lại snapshot không?
* Event có idempotent không?

---

## 3.4 Fallback to Polling

### Bản chất

> **WebSocket không phải lúc nào cũng available**

Có thể fail vì:

* Corporate proxy
* Firewall
* Network policy
* Legacy environment

---

### Fallback là gì?

* Nếu WS fail → dùng polling / long-polling
* Trải nghiệm giảm nhưng **không chết app**

---

### Trade-off

| WebSocket      | Polling        |
| -------------- | -------------- |
| Low latency    | Higher latency |
| Push           | Pull           |
| Phức tạp       | Đơn giản       |
| Tốn connection | Tốn request    |

---

### Nguyên tắc thực tế (Principal)

> **Realtime quan trọng với UX, nhưng availability quan trọng hơn**

Fallback không phải optional trong app lớn.

---

## 3.5 WebSocket Mental Checklist

Trước khi dùng WebSocket, tự hỏi:

* Có thật sự cần server push không?
* Tần suất update bao nhiêu?
* Connection scale bao nhiêu user?
* Mất connection thì UX ra sao?
* Fallback đã nghĩ chưa?

## 3.6 Message Protocols & Patterns

### Bản chất

> **WebSocket chỉ transport, không có structure**

Raw WebSocket = gửi string/binary
Cần protocol layer để structured communication

---

### Common patterns

**1. JSON-RPC style**
```
{
  "type": "request",
  "id": "123",
  "method": "sendMessage",
  "params": {...}
}
```

**2. Event-based**
```
{
  "event": "message.new",
  "data": {...}
}
```

**3. Command pattern**
```
{
  "cmd": "subscribe",
  "channel": "chat:room1"
}
```

---

### Best practices

* Version your message format
* Include message ID for tracking
* Type field cho routing
* Timestamp cho debugging

---

## 3.7 Scaling WebSocket Servers

### Bản chất

> **WebSocket = stateful = khó scale horizontal**

---

### Challenges

* Sticky sessions required
* State không share giữa servers
* Broadcast message phức tạp

---

### Solutions

**Redis Pub/Sub**
* Server A nhận message → publish Redis
* All servers subscribe → forward to clients

**Message Queue**
* RabbitMQ, Kafka
* Reliable, persistent

**Dedicated WS Gateway**
* Tách WS server khỏi API server
* Gateway chỉ làm routing

---

### Metrics cần track

* Active connections
* Messages/second
* Connection churn rate
* Average connection lifetime
* Memory per connection (~3-5KB)

---

## 3.8 Security Considerations

### CSRF Protection

> **WebSocket không có same-origin policy như HTTP**

---

### Best practices

* Validate Origin header
* Require auth token in handshake
* Rate limit connections per IP/user
* Timeout idle connections
* Validate every message

---

### Anti-patterns

❌ Accept connection từ any origin
❌ Trust client message không validate
❌ Không có message size limit

---

## 3.9 Testing WebSocket

### Bản chất

> **WebSocket integration tests khó hơn REST**

---

### Test cases cần có

* Connection lifecycle
* Reconnection logic
* Message ordering
* Concurrent messages
* Network failure simulation
* Load testing (concurrent connections)

---

### Tools

* WebSocket client libraries (ws, socket.io-client)
* Artillery, k6 for load testing
* Mock WebSocket server cho unit tests

---

