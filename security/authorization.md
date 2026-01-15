# Authorization – Complete Guide (Concept-first, Production-ready)

---

## 0. Authorization là gì? (Hiểu đúng ngay từ đầu)

**Authorization = Quyết định một hành động có được phép hay không**

> Trả lời câu hỏi: **“Với identity này, action này có được phép thực hiện trên resource này không?”**

So sánh nhanh:

* Authentication: *Bạn là ai?*
* Authorization: *Bạn được phép làm gì, trên cái gì, trong ngữ cảnh nào?*

⚠️ Nguyên tắc vàng:

> **Authorization luôn là business logic**
> Không phải UI logic, không phải FE logic

---

## 1. Mental Model Chuẩn của Authorization

Một quyết định authorization **luôn có đủ 3 thành phần**:

```
Subject   (Ai)      → user
Action    (Làm gì)  → read / write / delete
Resource  (Trên cái gì) → document / project / invoice
```

Ví dụ:

> “User A có được **edit** **document X** hay không?”

Nếu thiếu 1 trong 3 → authorization **chưa đầy đủ**.

---

## 2. Nơi Authorization PHẢI xảy ra

### Backend là source of truth

* FE **chỉ để UX** (ẩn/hiện nút)
* Backend **mới là nơi enforce quyền**

❌ Sai lầm phổ biến:

* Chặn button ở FE nhưng API không check
* Tin vào role FE gửi lên

> **FE bị hack được, backend thì không được phép sai**

---

## 3. Role-Based Access Control (RBAC)

### RBAC là gì?

**RBAC = Quyền được gán theo vai trò**

Ví dụ:

* Admin
* Editor
* Viewer

User → Role → Quyền

---

### Khi nào RBAC hoạt động tốt?

RBAC phù hợp khi:

* Tổ chức có **cấu trúc rõ ràng**
* Số role **ít và ổn định**

Ví dụ:

* Internal admin tool
* CMS đơn giản

---

### Hạn chế bản chất của RBAC

RBAC **fail khi scale**, vì:

* Role tăng theo tổ hợp quyền
* Business rule phức tạp không map được

Ví dụ xấu:

```
Admin
Admin_ReadOnly
Admin_Limited
Admin_RegionA
```

> Nếu role bắt đầu chứa logic → RBAC đang bị lạm dụng

---

## 4. Permission-based Authorization

### Permission là gì?

**Permission = một hành động cụ thể trên một loại resource**

Ví dụ:

* `project.read`
* `project.edit`
* `invoice.approve`

User → Permission

---

### Vì sao Permission-based scale tốt hơn?

* Permission **ổn định hơn role**
* Business thay đổi → chỉ map lại

Trade-off:

* Hệ thống phức tạp hơn
* Cần tooling & convention rõ ràng

---

## 5. Ownership-based Authorization

### Ownership là gì?

> *“User chỉ được thao tác trên resource của chính mình”*

Ví dụ:

* Sửa profile cá nhân
* Edit bài viết do mình tạo

Authorization rule:

```
resource.owner_id === user.id
```

⚠️ Ownership **không phải role**, là **ngữ cảnh**

---

## 6. Route Protection (Frontend)

### Route protection dùng để làm gì?

* Tránh user vào nhầm màn hình
* Cải thiện UX

Ví dụ:

* Chưa login → redirect /login
* Không phải admin → ẩn admin route

---

### Route protection KHÔNG phải security

❌ Không thay thế backend authorization
❌ Không được coi là enforcement

> Route protection = **UX optimization**, không phải security layer

---

## 7. API Authorization (Backend)

### API Authorization là gì?

> Kiểm tra quyền **ở mỗi request**

Flow chuẩn:

```
Request → Authenticate → Authorize → Business Logic
```

Ví dụ logic:

* Verify token → lấy user
* Check permission / ownership
* Cho phép hoặc reject (403)

---

## 7.1 Frontend Authorization Implementation

### Pattern 1: Route Guards (React Router v6)

```javascript
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from './AuthContext';

function ProtectedRoute({ children, requiredPermission, requiredRole }) {
  const { user, permissions, loading } = useAuth();
  const location = useLocation();
  
  if (loading) {
    return <LoadingSpinner />;
  }
  
  // Not authenticated
  if (!user) {
    // Save intended destination
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  
  // Check role if required
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/403" replace />;
  }
  
  // Check permission if required
  if (requiredPermission && !permissions.includes(requiredPermission)) {
    return <Navigate to="/403" replace />;
  }
  
  return children;
}

// Usage in App.jsx
function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      
      <Route path="/dashboard" element={
        <ProtectedRoute>
          <Dashboard />
        </ProtectedRoute>
      } />
      
      <Route path="/admin" element={
        <ProtectedRoute requiredRole="admin">
          <AdminPanel />
        </ProtectedRoute>
      } />
      
      <Route path="/documents/:id/edit" element={
        <ProtectedRoute requiredPermission="document.edit">
          <DocumentEditor />
        </ProtectedRoute>
      } />
      
      <Route path="/403" element={<Forbidden />} />
    </Routes>
  );
}
```

---

### Pattern 2: Permission Hooks

```javascript
// hooks/usePermission.js
export function usePermission(permission) {
  const { permissions } = useAuth();
  return permissions?.includes(permission) || false;
}

export function usePermissions(requiredPermissions) {
  const { permissions } = useAuth();
  
  // Check if user has ALL required permissions
  return requiredPermissions.every(p => permissions?.includes(p));
}

// Usage
function DocumentActions({ document }) {
  const { user } = useAuth();
  const canEdit = usePermission('document.edit');
  const canDelete = usePermission('document.delete');
  
  const isOwner = document.ownerId === user.id;
  
  return (
    <div>
      {(canEdit && isOwner) && <EditButton />}
      {canDelete && <DeleteButton />}
    </div>
  );
}
```

---

### Pattern 3: Optimistic UI + Server Validation

```javascript
function DocumentList() {
  const [documents, setDocuments] = useState([]);
  const canDelete = usePermission('document.delete');
  
  const handleDelete = async (documentId) => {
    // 1. Optimistic update
    const previous = documents;
    setDocuments(docs => docs.filter(d => d.id !== documentId));
    
    try {
      // 2. Server validates
      await api.delete(`/documents/${documentId}`);
      toast.success('Deleted');
      
    } catch (error) {
      // 3. Revert on 403
      if (error.status === 403) {
        setDocuments(previous);
        toast.error('Permission denied');
      }
    }
  };
  
  return (
    <div>
      {documents.map(doc => (
        <div key={doc.id}>
          {canDelete && <Button onClick={() => handleDelete(doc.id)}>Delete</Button>}
        </div>
      ))}
    </div>
  );
}
```

---

## 7.2 Permission Naming Convention

### ❌ Bad:
```
user_read
user_write
UserDelete
INVOICE_APPROVE
```

### ✅ Good:
```
resource.action

Examples:
- document.read
- document.create
- document.update
- document.delete
- invoice.approve
- user.impersonate
```

### Granularity Levels:

**Level 1: Coarse (simple apps)**
```
users.manage
documents.manage
```

**Level 2: CRUD (most apps)**
```
users.read
users.create
users.update
users.delete
```

**Level 3: Fine-grained (complex apps)**
```
users.read.own
users.read.department
users.read.all
users.update.profile
users.update.roles
```

### Wildcard Pattern:
```
users.*        → All user permissions
*.read         → Read all resources
*.*            → Superadmin (use sparingly!)
```

---

## 7.3 Backend Authorization Patterns

### Pattern 1: Middleware-based

```javascript
// Express.js
function requirePermission(permission) {
  return async (req, res, next) => {
    const user = req.user; // From auth middleware
    
    if (!user.permissions.includes(permission)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    next();
  };
}

// Usage
app.delete('/documents/:id', 
  requirePermission('document.delete'),
  deleteDocument
);
```

### Pattern 2: Resource-level Check

```javascript
async function updateDocument(req, res) {
  const document = await Document.findById(req.params.id);
  
  // Ownership check
  if (document.ownerId !== req.user.id && !req.user.isAdmin) {
    return res.status(403).json({ error: 'Not your document' });
  }
  
  // Permission check
  if (!req.user.permissions.includes('document.update')) {
    return res.status(403).json({ error: 'No permission' });
  }
  
  await document.update(req.body);
  res.json(document);
}
```

### Pattern 3: Policy Functions

```javascript
// policies/documentPolicy.js
const DocumentPolicy = {
  canUpdate(user, document) {
    if (user.isAdmin) return true;
    if (document.ownerId === user.id) return true;
    if (user.role === 'editor' && user.permissions.includes('document.update')) {
      return true;
    }
    return false;
  }
};

// Usage
if (!DocumentPolicy.canUpdate(req.user, document)) {
  return res.status(403).json({ error: 'Forbidden' });
}
```

---

## 8. Hybrid Model (RBAC + Permission + Ownership)

### Vì sao hybrid là thực tế nhất?

Hệ thống thực tế thường:

* Dùng role cho **macro access**
* Dùng permission cho **chi tiết**
* Dùng ownership cho **resource cụ thể**

Ví dụ:

* Admin role
* Có `project.edit` permission
* Chỉ edit project thuộc tenant mình

---

## 9. Authorization Policy ≠ If / Else

### Sai lầm kinh điển

```text
if (isAdmin) allow
else if (isOwner) allow
else deny
```

Vấn đề:

* Logic rải rác
* Khó audit
* Khó test

---

### Authorization đúng là Policy

> Authorization nên được viết như **rule**, không phải code flow

Ví dụ tư duy:

```
ALLOW if
- user has permission X
- AND resource thuộc tenant Y
```

---

## 10. ABAC (Attribute-Based Access Control)

### ABAC là gì?

**ABAC = Authorization dựa trên attributes (properties)**

Authorization decision dựa trên:
- **Subject attributes:** user properties (role, department, clearance level)
- **Resource attributes:** document properties (classification, owner, created date)
- **Environment attributes:** context (time, location, IP address)
- **Action:** operation being performed

---

### Khi nào cần ABAC?

RBAC/Permission không đủ khi business rules phức tạp:

**Example scenarios:**

❌ **RBAC không express được:**
> "Managers can approve invoices < $10,000 in their department during business hours"

✅ **ABAC express được:**
```javascript
const rule = {
  subject: {
    role: 'manager',
    department: user.department
  },
  action: 'approve',
  resource: {
    type: 'invoice',
    amount: { $lt: 10000 },
    department: user.department // Same department
  },
  environment: {
    time: { between: ['09:00', '17:00'] },
    day: { in: ['Mon', 'Tue', 'Wed', 'Thu', 'Fri'] }
  }
};
```

---

### ABAC Implementation Pattern

**Backend (Policy evaluation):**

```javascript
// policies/invoicePolicy.js
class InvoicePolicy {
  static canApprove(user, invoice, context) {
    // Subject attributes
    const isManager = user.role === 'manager';
    const sameDepartment = user.department === invoice.department;
    
    // Resource attributes
    const isUnderLimit = invoice.amount < 10000;
    const isPending = invoice.status === 'pending';
    
    // Environment attributes
    const now = new Date();
    const isBusinessHours = 
      now.getHours() >= 9 && 
      now.getHours() < 17 &&
      now.getDay() >= 1 && 
      now.getDay() <= 5;
    
    // Combine all conditions
    return (
      isManager &&
      sameDepartment &&
      isUnderLimit &&
      isPending &&
      isBusinessHours
    );
  }
}

// API endpoint
app.post('/invoices/:id/approve', async (req, res) => {
  const invoice = await Invoice.findById(req.params.id);
  
  if (!InvoicePolicy.canApprove(req.user, invoice, { time: new Date() })) {
    return res.status(403).json({ 
      error: 'Cannot approve this invoice',
      reasons: getFailureReasons(req.user, invoice)
    });
  }
  
  await invoice.approve(req.user.id);
  res.json(invoice);
});
```

---

### ABAC Libraries

**1. Casbin (Recommended)**
```javascript
const { newEnforcer } = require('casbin');

// Define policy model
const model = `
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub.role == p.sub && r.obj.type == p.obj && r.act == p.act && r.obj.amount < 10000
`;

// Policy rules
const policy = `
p, manager, invoice, approve
`;

// Enforce
const enforcer = await newEnforcer(model, policy);

const allowed = await enforcer.enforce({
  role: user.role,
  department: user.department
}, {
  type: 'invoice',
  amount: invoice.amount,
  department: invoice.department
}, 'approve');
```

**2. oso (Policy-as-code)**
```polar
# policies/invoice.polar

# Managers can approve invoices in their department
allow(user: User, "approve", invoice: Invoice) if
  user.role = "manager" and
  user.department = invoice.department and
  invoice.amount < 10000;

# Admins can approve all invoices
allow(user: User, "approve", invoice: Invoice) if
  user.role = "admin";
```

```javascript
const { Oso } = require('oso');

const oso = new Oso();
await oso.loadFiles(['policies/invoice.polar']);

const allowed = await oso.isAllowed(user, 'approve', invoice);
```

---

### ABAC vs RBAC Decision Matrix

| Scenario | Use RBAC | Use ABAC |
|----------|----------|----------|
| Simple role hierarchy | ✅ | ❌ |
| Static permissions | ✅ | ❌ |
| Same-department restrictions | ❌ | ✅ |
| Time-based access | ❌ | ✅ |
| Amount/value thresholds | ❌ | ✅ |
| Multi-tenant complex rules | ❌ | ✅ |
| Compliance requirements (GDPR, HIPAA) | ❌ | ✅ |

**Rule of thumb:**
- Start with RBAC
- Add ABAC khi RBAC không express được business rules
- Hybrid approach (RBAC for macro, ABAC for fine-grained)

---

## 11. Những Anti-pattern phổ biến

## 11. Những Anti-pattern phổ biến

### ❌ Anti-pattern 1: Nhét role vào JWT payload dài hạn

**Problem:**
```javascript
// JWT payload
{
  userId: 123,
  role: 'admin', // ❌ Hardcoded in long-lived token
  exp: 1735689600 // Expires in 30 days
}
```

**Tại sao sai:**
- User bị demote từ admin → phải đợi token expire
- Role change không reflect ngay
- Security risk cao

**Fix:**
```javascript
// ✅ JWT chỉ chứa userId
{
  userId: 123,
  exp: 1735689600
}

// ✅ Fetch fresh permissions mỗi request
async function getCurrentUser(userId) {
  const user = await db.users.findById(userId);
  return {
    ...user,
    role: user.role, // Fresh from DB
    permissions: await fetchUserPermissions(userId) // Real-time
  };
}
```

---

### ❌ Anti-pattern 2: FE quyết định quyền thật

**Problem:**
```javascript
// ❌ Frontend logic
function deleteDocument(id) {
  if (user.role === 'admin') {
    // Direct API call, no backend check
    api.delete(`/documents/${id}`);
  }
}
```

**Fix:**
```javascript
// ✅ Always validate on backend
// Frontend: UX optimization only
function deleteDocument(id) {
  // Show button based on permission (UX)
  api.delete(`/documents/${id}`); // Backend validates
}

// Backend: Real enforcement
app.delete('/documents/:id', (req, res) => {
  if (!req.user.permissions.includes('document.delete')) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  // Proceed
});
```

---

### ❌ Anti-pattern 3: Mỗi API tự viết logic khác nhau

**Problem:**
```javascript
// ❌ Inconsistent authorization logic
app.delete('/documents/:id', (req, res) => {
  if (req.user.role !== 'admin') return res.status(403);
  // ...
});

app.delete('/invoices/:id', (req, res) => {
  if (!req.user.permissions.includes('delete')) return res.status(403);
  // ...
});

app.delete('/users/:id', (req, res) => {
  if (req.user.id !== req.params.id) return res.status(403);
  // ...
});
```

**Fix:**
```javascript
// ✅ Centralized policy
const DocumentPolicy = require('./policies/DocumentPolicy');
const InvoicePolicy = require('./policies/InvoicePolicy');

app.delete('/documents/:id', (req, res) => {
  const doc = await Document.findById(req.params.id);
  if (!DocumentPolicy.canDelete(req.user, doc)) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  // ...
});
```

---

### ❌ Anti-pattern 4: Không log authorization decisions

**Problem:**
Authorization failures thất bại nhưng không biết tại sao.

**Fix:**
```javascript
// ✅ Log authorization events
function checkPermission(user, permission, resource) {
  const allowed = user.permissions.includes(permission);
  
  // Log decision
  logger.info('Authorization check', {
    userId: user.id,
    permission,
    resourceType: resource.type,
    resourceId: resource.id,
    allowed,
    timestamp: new Date()
  });
  
  if (!allowed) {
    logger.warn('Authorization denied', {
      userId: user.id,
      permission,
      resource: resource.id
    });
  }
  
  return allowed;
}
```

**Benefits:**
- Audit trail
- Debug authorization issues
- Detect attack patterns
- Compliance requirements

---

### ❌ Anti-pattern 5: Test authorization bằng tay

**Problem:**
Authorization logic phức tạp, test manual = miss cases.

**Fix:**
```javascript
// ✅ Automated tests
describe('Document Authorization', () => {
  it('owner can edit own document', async () => {
    const owner = { id: 1, role: 'user', permissions: ['document.edit'] };
    const document = { id: 100, ownerId: 1 };
    
    expect(DocumentPolicy.canEdit(owner, document)).toBe(true);
  });
  
  it('non-owner cannot edit document', async () => {
    const user = { id: 2, role: 'user', permissions: ['document.edit'] };
    const document = { id: 100, ownerId: 1 };
    
    expect(DocumentPolicy.canEdit(user, document)).toBe(false);
  });
  
  it('admin can edit any document', async () => {
    const admin = { id: 3, role: 'admin', permissions: ['document.edit'] };
    const document = { id: 100, ownerId: 1 };
    
    expect(DocumentPolicy.canEdit(admin, document)).toBe(true);
  });
});
```

---

## 12. Authorization Review Checklist

### 📋 Frontend Checklist

**Route Protection:**
- ☐ Protected routes có `ProtectedRoute` wrapper
- ☐ Route guards check authentication trước authorization
- ☐ 403 page exists và user-friendly
- ☐ Redirect sau login về intended page

**Component-level:**
- ☐ Permission checks dùng hooks (reusable)
- ☐ Buttons/actions ẩn khi không có quyền (UX)
- ☐ Disabled state có tooltip explaining why
- ☐ Optimistic UI có revert mechanism

**Error Handling:**
- ☐ 403 errors được handle globally
- ☐ User-friendly error messages
- ☐ No sensitive info leaked trong error

---

### 📋 Backend Checklist

**Authorization Logic:**
- ☐ Every endpoint có authorization check
- ☐ Authorization logic centralized (policies/middleware)
- ☐ Permission naming convention consistent
- ☐ Ownership checks khi cần thiết

**Security:**
- ☐ No authorization logic chỉ ở frontend
- ☐ Role/permission fresh từ DB, không cached lâu
- ☐ Authorization failures được log
- ☐ Rate limiting on sensitive endpoints

**Testing:**
- ☐ Unit tests cho policy functions
- ☐ Integration tests cho authorization flows
- ☐ Test unauthorized access attempts
- ☐ Test edge cases (ownership, multi-tenant)

---

### 📋 Architecture Checklist

**Design:**
- ☐ Authorization model documented (RBAC/ABAC/Hybrid)
- ☐ Permission list maintained và versioned
- ☐ Role hierarchy clear (nếu có)
- ☐ Ownership rules explicit

**Scalability:**
- ☐ Authorization không block request flow
- ☐ Permission caching strategy (nếu cần)
- ☐ Multi-tenant isolation enforced
- ☐ Audit trail cho compliance

---

## 13. Production Considerations

### 🔒 Security Best Practices

**1. Defense in Depth**
```
Frontend check (UX) 
  → API Gateway check (optional)
    → Backend endpoint check (required)
      → Database row-level security (optional)
```

**2. Principle of Least Privilege**
- User chỉ có permissions cần thiết
- Default = deny, explicitly allow
- Temporary elevated permissions có expiry

**3. Separation of Concerns**
- Authentication ≠ Authorization
- Route protection ≠ Security
- Frontend check ≠ Enforcement

---

### 📊 Monitoring & Auditing

**Log critical events:**
```javascript
// Authorization decision
logger.info('authz', { user, action, resource, allowed });

// Permission changes
logger.info('permission_granted', { userId, permission, grantedBy });
logger.info('permission_revoked', { userId, permission, revokedBy });

// Role changes
logger.info('role_changed', { userId, oldRole, newRole, changedBy });
```

**Metrics to track:**
- 403 error rate (spike = attack hoặc permission issue)
- Authorization check latency
- Permission cache hit rate
- Failed authorization attempts per user

---

## 14. Tổng kết

### Core Principles
* Authorization = **decision**, không phải UI logic
* **Backend enforcement** bắt buộc, frontend chỉ UX
* RBAC đơn giản nhưng không scale cho complex rules
* Permission-based là foundation dài hạn
* ABAC cho business logic phức tạp

### Implementation Essentials
* Route guards cho navigation protection
* Permission hooks cho component-level checks
* Optimistic UI + server validation
* Centralized policy functions
* Comprehensive logging & auditing

### Architecture Rules
* Start simple (RBAC), evolve khi cần (ABAC)
* Permission naming convention matters
* Test authorization thoroughly
* Monitor 403 errors
* Document permission model

> **Nguyên tắc vàng**:
> *Authorization sai → security fail dù authentication đúng*

> **Production reality**:
> *Authorization logic chiếm 30-40% backend code trong enterprise apps*

---

**DOCUMENT UPDATED ✅**

*Version: 2.0*  
*Last updated: 2026-01-03*  
*Added: Frontend patterns, ABAC, Backend patterns, Permission naming, Comprehensive checklist*
