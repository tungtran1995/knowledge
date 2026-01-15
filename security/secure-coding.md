# 5. Secure Coding (The Last Line of Defense)

> **Secure Coding không phải là 1 kỹ thuật**
> mà là **thái độ & kỷ luật khi viết code**, ngay cả khi không ai nhắc đến security.

Nếu ví security như hệ thống phòng thủ thì:

* Auth / Authz = Cổng chính
* Mitigation = Tường, camera, khóa
* **Secure Coding = thói quen của người sống trong nhà**

---

## 5.1 Never Trust User Input

### Bản chất

> **User input = dữ liệu không đáng tin cho đến khi được chứng minh ngược lại**

“User” không chỉ là:

* Người dùng thật
* Mà là:

  * Script
  * Bot
  * Attacker
  * Tool tự động

---

### User input là gì?

* Form data
* Query params
* Headers
* Cookie
* Request body
* Data từ API khác

👉 **Mọi thứ đi vào hệ thống đều là user input**

---

### Sai lầm phổ biến

* Tin rằng FE đã validate rồi
* Tin rằng API nội bộ là an toàn
* Tin rằng user “không làm vậy đâu”

> Attacker không dùng UI của bạn.

---

### Nguyên tắc chuẩn

```
Validate early
Validate strictly
Validate again at boundary
```

* FE validate → UX
* BE validate → Security

---

## 5.2 Never Store Secrets in Frontend

### Bản chất

> **Frontend là môi trường không kiểm soát được**

* Source code public
* Bundle bị đọc được
* Runtime bị inspect được

👉 **Không có khái niệm “secret” ở frontend**

---

### Những thứ KHÔNG BAO GIỜ được để ở FE

* API secret
* Private key
* Database credential
* OAuth client secret

---

### Sai lầm hay gặp

* `.env` bị bundle vào JS
* “Chỉ để tạm”
* “Không ai biết đâu”

> Nếu FE cần secret → thiết kế đã sai.

---

### Nguyên tắc

* FE chỉ giữ:

  * Public identifier
  * Temporary token (short-lived)
* Secret luôn ở:

  * Backend
  * Secure storage

---

## 5.3 Use HTTPS Only

### Bản chất

> **HTTPS không chỉ để encrypt — mà để xác thực & toàn vẹn**

HTTPS đảm bảo:

* Không bị đọc trộm
* Không bị sửa giữa đường
* Server là server thật

---

### HTTP nguy hiểm ở đâu?

* MITM attack
* Token bị leak
* JS bị inject

👉 HTTP = mất toàn bộ security assumptions phía trên

---

### Nguyên tắc chuẩn

* HTTPS everywhere
* Redirect HTTP → HTTPS
* HSTS nếu có thể

> Security layer khác **vô nghĩa** nếu không có HTTPS.

---

## 5.4 Regular Dependency Updates

### Bản chất

> **Dependency = code của người khác chạy trong hệ thống của bạn**

* Bạn không kiểm soát nó
* Nhưng bạn chịu trách nhiệm khi nó bị exploit

---

### Sai lầm phổ biến

* “Không ai dùng feature đó”
* “Lib này nhỏ mà”
* “Update sợ break”

👉 Attacker không quan tâm bạn có dùng hay không.

---

### Nguyên tắc thực tế

* Lock version
* Audit định kỳ
* Update có kiểm soát

> Dependency security = hygiene, không phải feature.

---

## 5.4.1 Password Hashing (Critical Secure Coding Pattern)

### Bản chất

> **NEVER store passwords in plain text**

Password = most sensitive user data.  
If db compromised → hashed passwords give time to notify users.

---

### ❌ Anti-patterns

```javascript
// ❌ NEVER: Plain text password
db.users.insert({
  username: 'alice',
  password: 'MyPassword123' // Catastrophic!
});

// ❌ NEVER: Simple hash (MD5, SHA1)
const crypto = require('crypto');
const hash = crypto.createHash('md5').update(password).digest('hex');
// MD5/SHA1 too fast → rainbow table attacks

// ❌ NEVER: Hash without salt
const hash = crypto.createHash('sha256').update(password).digest('hex');
// Same password → same hash → pre-computed attacks
```

---

### ✅ Correct Approach: bcrypt

```javascript
const bcrypt = require('bcrypt');

// Registration
async function registerUser(username, password) {
  // Generate salt + hash (10 rounds = good balance)
  const saltRounds = 10;
  const hashedPassword = await bcrypt.hash(password, saltRounds);
  
  await db.users.insert({
    username,
    password: hashedPassword // Store hash, not plain text
  });
}

// Login
async function loginUser(username, password) {
  const user = await db.users.findOne({ username });
  
  if (!user) {
    return { success: false, error: 'Invalid credentials' };
  }
  
  // Compare plain password với stored hash
  const match = await bcrypt.compare(password, user.password);
  
  if (match) {
    return { success: true, user };
  } else {
    return { success: false, error: 'Invalid credentials' };
  }
}
```

**Why bcrypt:**
- Slow by design (prevent brute force)
- Auto salt (different hash cho same password)
- Adjustable cost factor (increase as hardware improves)

---

### Alternative: Argon2 (Modern, recommended)

```javascript
const argon2 = require('argon2');

// Hash password
async function hashPassword(password) {
  const hash = await argon2.hash(password);
  return hash;
}

// Verify password
async function verifyPassword(hash, password) {
  try {
    return await argon2.verify(hash, password);
  } catch (err) {
    return false;
  }
}
```

**Argon2 advantages:**
- Winner of Password Hashing Competition (2015)
- Resistant to GPU/ASIC attacks
- Memory-hard (expensive to crack)

---

## 5.4.2 Error Handling Without Information Leakage

### Bản chất

> **Errors expose system internals**

Detailed errors help developers debug.  
Detailed errors help attackers exploit.

---

### ❌ Information Leakage

```javascript
// ❌ Expose stack trace to client
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await db.query('SELECT * FROM users WHERE id = ?', [req.params.id]);
    res.json(user);
  } catch (error) {
    // Leak internal error
    res.status(500).json({
      error: error.message, // "Users table doesn't exist"
      stack: error.stack     // Full file paths, line numbers
    });
  }
});

// Attacker learns:
// - Database structure
// - File paths
// - Framework version
// - Potential vulnerabilities
```

---

### ✅ Secure Error Handling

```javascript
// ✅ Generic error to client, detailed log for devs
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await db.query('SELECT * FROM users WHERE id = ?', [req.params.id]);
    res.json(user);
  } catch (error) {
    // Log detailed error internally
    logger.error('Database query failed', {
      error: error.message,
      stack: error.stack,
      userId: req.params.id,
      timestamp: new Date()
    });
    
    // Generic error to client
    res.status(500).json({
      error: 'Internal server error' // No details
    });
  }
});
```

---

### Pattern: Error Response Levels

```javascript
const isDevelopment = process.env.NODE_ENV === 'development';

function errorResponse(error, req, res) {
  // Log full error (always)
  logger.error({
    message: error.message,
    stack: error.stack,
    url: req.url,
    method: req.method,
    body: req.body
  });
  
  if (isDevelopment) {
    // Development: Full error details
    return res.status(500).json({
      error: error.message,
      stack: error.stack
    });
  } else {
    // Production: Generic message only
    return res.status(500).json({
      error: 'An error occurred. Please try again later.'
    });
  }
}
```

---

### Login Error Message (Security-sensitive)

```javascript
// ❌ Leak user existence
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  
  const user = await db.users.findOne({ username });
  
  if (!user) {
    return res.status(401).json({
      error: 'User not found' // Attacker knows user doesn't exist
    });
  }
  
  const match = await bcrypt.compare(password, user.password);
  
  if (!match) {
    return res.status(401).json({
      error: 'Incorrect password' // Attacker knows user exists
    });
  }
});

// ✅ Generic message (don't leak user existence)
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  
  const user = await db.users.findOne({ username });
  
  if (!user || !(await bcrypt.compare(password, user.password))) {
    // Same message for both cases
    return res.status(401).json({
      error: 'Invalid credentials'
    });
  }
  
  // Success
  res.json({ token: generateToken(user) });
});
```

---

## 5.4.3 Logging Sensitive Data (What NOT to log)

### Bản chất

> **Logs = long-term storage, often less protected than database**

---

### ❌ Logging sensitive data

```javascript
// ❌ NEVER log passwords
logger.info('User login attempt', {
  username: req.body.username,
  password: req.body.password // Catastrophic!
});

// ❌ NEVER log full credit cards
logger.info('Payment processed', {
  cardNumber: '4532-1234-5678-9010' // PCI violation
});

// ❌ NEVER log session tokens
logger.info('API request', {
  authToken: req.headers.authorization // Steal-able
});
```

---

### ✅ Secure logging

```javascript
// ✅ Log username, NOT password
logger.info('User login attempt', {
  username: req.body.username,
  ip: req.ip,
  userAgent: req.headers['user-agent']
  // password: NEVER
});

// ✅ Mask credit card (last 4 digits only)
const maskedCard = card.replace(/\d(?=\d{4})/g, '*');
logger.info('Payment processed', {
  cardNumber: maskedCard // ****-****-****-9010
});

// ✅ Log token hash or ID, not token itself
const tokenHash = crypto.createHash('sha256')
  .update(req.headers.authorization)
  .digest('hex');

logger.info('API request', {
  tokenHash: tokenHash.substring(0, 8), // First 8 chars of hash
  userId: req.user.id
});
```

---

### Redaction Pattern

```javascript
function redactSensitiveData(obj) {
  const sensitiveKeys = ['password', 'token', 'secret', 'creditCard', 'ssn'];
  
  const redacted = { ...obj };
  
  for (const key in redacted) {
    if (sensitiveKeys.some(k => key.toLowerCase().includes(k))) {
      redacted[key] = '[REDACTED]';
    }
  }
  
  return redacted;
}

// Usage
logger.info('User data', redactSensitiveData(req.body));
// Output: { username: 'alice', password: '[REDACTED]' }
```

---

## 5.5 Security Headers

### Bản chất

> **Security headers là contract giữa server & browser**

Chúng nói:

* Browser **được phép làm gì**
* Và **không được phép làm gì**

---

### Những header quan trọng (concept-level)

* CSP → hạn chế JS & resource
* X-Frame-Options → chống clickjacking
* HSTS → ép HTTPS
* Referrer-Policy → tránh leak info

👉 Header **không fix bug**, nhưng:

* Giảm impact
* Giảm exploitability

---

### Sai lầm phổ biến

* Copy config không hiểu
* Tắt CSP vì “dev khó”
* Coi header là optional

> Header là **defense-in-depth**, không phải decoration.

---

## 5.6 Secure Coding là gì, nói cho gọn?

> **Secure Coding = Viết code với giả định rằng hệ thống sẽ bị tấn công**

Không phải:

* Viết code phức tạp hơn
* Viết code paranoid

Mà là:

* Không tin input
* Không tin client
* Không tin network
* Không tin dependency

---

## 5.7 Comprehensive Secure Coding Checklist

### 📋 Input Handling
- ☐ **Never trust any input** (forms, URL, headers, cookies, API)
- ☐ Validate format (regex, length, type)
- ☐ Sanitize before storage (DOMPurify for HTML)
- ☐ Encode before output (framework default or explicit)
- ☐ Validate BOTH client + server (client = UX, server = security)

**Example checklist item:**
```javascript
// ☐ Input validated?
const { username } = req.body;
if (!/^[a-zA-Z0-9_]{3,20}$/.test(username)) {
  return res.status(400).json({ error: 'Invalid format' });
}
```

---

### 🔐 Authentication/Authorization
- ☐ Never store passwords in plain text (bcrypt/argon2)
- ☐ Hash passwords with salt (auto in bcrypt)
- ☐ Short-lived access tokens (<15m)
- ☐ HttpOnly cookies for refresh tokens
- ☐ Check authorization on EVERY endpoint
- ☐ Verify ownership, not just role

**Example:**
```javascript
// ☐ Authorization checked?
if (document.ownerId !== req.user.id && !req.user.isAdmin) {
  return res.status(403).json({ error: 'Forbidden' });
}
```

---

### 🛡️ Output/Rendering
- ☐ Use framework auto-escaping (React `{}`, Vue `{{}}`)
- ☐ AVOID `dangerouslySetInnerHTML` / `v-html` unless sanitized
- ☐ Use `textContent` not `innerHTML` for plain text
- ☐ NEVER use `eval()` or `new Function()` with user input
- ☐ Validate URLs before redirecting

---

### 🗄️ Database
- ☐ Use parameterized queries or ORM (NEVER string concatenation)
- ☐ Least privilege database accounts
- ☐ Hide database errors from clients (log internally)
- ☐ Implement rate limiting on queries

---

### 🌐 Network/API
- ☐ HTTPS everywhere (no HTTP)
- ☐ HSTS header enabled
- ☐ CSRF protection (SameSite cookies or token)
- ☐ CORS configured correctly (not `*` in production)
- ☐ Rate limiting on sensitive endpoints

---

### 📝 Logging
- ☐ NEVER log passwords
- ☐ NEVER log full tokens
- ☐ NEVER log full credit cards
- ☐ Mask sensitive data (last 4 digits only)
- ☐ Log security events (failed logins, auth errors)

---

### 📦 Dependencies
- ☐ Regular `npm audit` / `yarn audit`
- ☐ Lock dependency versions (`package-lock.json`)
- ☐ Update dependencies regularly (security patches)
- ☐ Remove unused dependencies
- ☐ Review dependency permissions

**Commands:**
```bash
# Check vulnerabilities
npm audit

# Fix auto-fixable vulnerabilities
npm audit fix

# Force update (breaking changes possible)
npm audit fix --force
```

---

### 🎯 Security Headers
- ☐ Content-Security-Policy (CSP)
- ☐ X-Frame-Options: DENY (clickjacking)
- ☐ X-Content-Type-Options: nosniff
- ☐ Referrer-Policy: no-referrer or strict-origin
- ☐ Permissions-Policy (formerly Feature-Policy)

---

## 5.8 Common Secure Coding Mistakes with Fixes

### ❌ Mistake 1: Using `eval()` or `new Function()`

```javascript
// ❌ NEVER
const userCode = req.query.code;
eval(userCode); // Arbitrary code execution!

// ✅ Don't allow dynamic code from user
// If absolutely needed, use sandboxed iframe or Web Worker
```

---

### ❌ Mistake 2: `innerHTML` với user input

```javascript
// ❌ XSS vulnerability
element.innerHTML = userInput;

// ✅ Use textContent
element.textContent = userInput;

// ✅ Or sanitize first
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userInput);
```

---

### ❌ Mistake 3: Regex Denial of Service (ReDoS)

```javascript
// ❌ Vulnerable regex (exponential backtracking)
const regex = /(a+)+b/;
const input = 'aaaaaaaaaaaaaaaaaaaaaaaac'; // No match, but...
// Takes exponential time → server hangs

// ✅ Safe regex (linear time)
const regex = /a+b/;

// ✅ Or use regex timeout (Node.js 16+)
try {
  const match = input.match(regex, { timeout: 1000 });
} catch (err) {
  // Timeout exceeded
}
```

---

### ❌ Mistake 4: Timing Attacks on Password Compare

```javascript
// ❌ Vulnerable to timing attack
function comparePasswords(input, stored) {
  if (input === stored) {
    return true;
  }
  return false;
}

// Attack: Measure response time
// Correct characters take slightly longer
// → Guess password character by character

// ✅ Constant-time comparison
const crypto = require('crypto');

function comparePasswords(input, stored) {
  const inputHash = crypto.createHash('sha256').update(input).digest();
  const storedHash = crypto.createHash('sha256').update(stored).digest();
  
  // timingSafeEqual prevents timing attacks
  return crypto.timingSafeEqual(inputHash, storedHash);
}

// ✅ Or use bcrypt.compare (already timing-safe)
const match = await bcrypt.compare(input, storedHash);
```

---

### ❌ Mistake 5: Predictable Random Values

```javascript
// ❌ Math.random() is NOT cryptographically secure
const sessionId = Math.random().toString(36); // Predictable!

// ✅ Use crypto.randomBytes
const crypto = require('crypto');
const sessionId = crypto.randomBytes(32).toString('hex');

// ✅ Or crypto.randomUUID (Node 16+)
const sessionId = crypto.randomUUID();
```

---

## 5.9 Code Review Security Prompts

**Before merging any PR, ask:**

```
Input:
❓ Dữ liệu này đến từ đâu?
❓ Có từ user không? (forms, URL, API)
❓ Đã validate format chưa?
❓ Đã sanitize/escape chưa?

Storage:
❓ Sensitive data có được hash/encrypt không? (passwords, tokens)
❓ Database query có dùng parameterized query không?
❓ Có string concatenation trong SQL không?

Output:
❓ Có render user input ra HTML không?
❓ Có dùng innerHTML/dangerouslySetInnerHTML không?
❓ Đã escape output chưa?

Authentication/Authorization:
❓ Endpoint này cần login không?
❓ Đã check permission chưa?
❓ Có verify ownership không? (resource thuộc user)
❓ Token có thể bị bypass không?

Side Effects:
❓ Action này change state không? (POST/PUT/DELETE)
❓ Có CSRF protection không?
❓ Có rate limiting không?

Dependencies:
❓ Có thêm dependency mới không?
❓ Đã check vulnerabilities chưa? (npm audit)
❓ Có alternative an toàn hơn không?
```

---

## 5.10 Kết nối với toàn bộ series

* **Authentication** → *Ai đang nói chuyện với hệ thống?*
* **Authorization** → *Họ được phép làm gì?*
* **Common Vulnerabilities** → *Hệ thống bị phá ở đâu?*
* **Prevention Strategies** → *Phòng thế nào cho đúng?*
* **Secure Coding** → *Làm sao không tự bắn vào chân mỗi ngày?*

---

## Kết luận cuối (rất quan trọng)

> **Security không fail vì thiếu kỹ thuật**  
> **mà vì thiếu kỷ luật lặp đi lặp lại**

### Secure Coding là gì?

**KHÔNG PHẢI:**
- Viết code phức tạp hơn
- Thêm nhiều layers không cần thiết
- Paranoid mọi thứ

**MÀ LÀ:**
- Validate input EVERY TIME
- Check authorization EVERY endpoint
- Hash passwords WITH salt
- Escape output BY DEFAULT
- Log security events ALWAYS
- Update dependencies REGULARLY

### Discipline > Technique

```
1 lần quên check authorization = 1 vulnerability
10 lần làm đúng + 1 lần quên = vẫn bị hack

→ Security = 100% consistency
```

### Checklist Usage

**Daily:**
- ☐ Validate user input
- ☐ Check authorization before actions
- ☐ Escape output

**Weekly:**
- ☐ Review security logs
- ☐ Check npm audit
- ☐ Review recent code changes

**Monthly:**
- ☐ Update dependencies (security patches)
- ☐ Security audit of new features
- ☐ Team security training

> *Secure coding chính là kỷ luật đó, được áp dụng mỗi ngày, trong từng dòng code.*

---

**DOCUMENT EXPANDED ✅**

*Version: 2.0*  
*Last updated: 2026-01-03*  
*Lines: 264 → ~600 (+127%)*  
*Added: Password hashing, Error handling, Logging patterns, Common mistakes, Comprehensive checklists*
