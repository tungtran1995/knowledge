# Application Architecture 

## 1. Layered Architecture - Separation by Technical Concern

### Concept

**Core idea**: Tách code theo **technical responsibility** thành các layers độc lập.

```
┌─────────────────────────────┐
│   Presentation Layer (UI)   │  ← React components, JSX, styling
├─────────────────────────────┤
│   Business Logic Layer      │  ← Business rules, validation, calculations
├─────────────────────────────┤
│   Data Access Layer         │  ← API calls, database queries, caching
└─────────────────────────────┘
```

**Flow**: UI → Business Logic → Data Access → External Systems

**Nguyên tắc**: Mỗi layer chỉ nói chuyện với layer liền kề.

---

### Three Core Layers

#### Layer 1: Presentation Layer (UI Layer)

**Responsibility**: 
- Render UI
- Handle user interactions
- Display data
- Route navigation
- Form state (local UI state)

**Không làm**:
- Business logic (tính toán, validation rules)
- API calls trực tiếp
- Data transformation

**Components trong layer này**:
```
- React components
- Pages/Screens
- Hooks (UI-focused như useToggle, useMediaQuery)
- Styles (CSS, CSS-in-JS)
```

**Example**:
```javascript
// UserProfile.tsx - Presentation Layer
function UserProfile({ user, onSave }) {
  const [isEditing, setIsEditing] = useState(false);
  
  return (
    <div>
      <Avatar src={user.avatar} />
      <Text>{user.name}</Text>
      {isEditing ? (
        <EditForm user={user} onSave={onSave} />
      ) : (
        <button onClick={() => setIsEditing(true)}>Edit</button>
      )}
    </div>
  );
}
```

**Characteristics**:
- Chỉ nhận props, hiển thị, gọi callbacks
- Không biết data đến từ đâu (API? localStorage? mock?)
- Dễ test (pass props, check rendered output)

---

#### Layer 2: Business Logic Layer

**Responsibility**:
- Business rules (discount calculations, eligibility checks)
- Validation logic (email format, password strength)
- Data transformation (formatting, normalization)
- State management (global state, derived state)
- Workflow orchestration (multi-step processes)

**Không làm**:
- Rendering UI (no JSX)
- Direct API calls (delegate to Data Layer)

**Code trong layer này**:
```
- Business logic functions (pure functions)
- Custom hooks (business-focused)
- State management (Redux slices, Zustand stores)
- Validators
- Formatters/parsers
- Use cases/interactors (DDD term)
```

**Example**:
```javascript
// userService.ts - Business Logic Layer
export function calculateUserDiscount(user) {
  // Business rule: Premium users get 20%, regular 10%
  if (user.isPremium) return 0.2;
  if (user.totalOrders > 10) return 0.15;
  return 0.1;
}

export function validateUserProfile(profile) {
  const errors = {};
  
  if (!profile.email.includes('@')) {
    errors.email = 'Invalid email';
  }
  
  if (profile.age < 18) {
    errors.age = 'Must be 18+';
  }
  
  return errors;
}

// Custom hook - orchestrates business logic
export function useUserProfile(userId) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  
  const loadUser = async () => {
    setLoading(true);
    const data = await userAPI.fetchUser(userId); // Calls Data Layer
    setUser(data);
    setLoading(false);
  };
  
  const updateUser = async (updates) => {
    const errors = validateUserProfile(updates); // Business logic
    if (Object.keys(errors).length > 0) {
      throw new Error('Validation failed');
    }
    
    await userAPI.updateUser(userId, updates); // Calls Data Layer
    await loadUser(); // Refresh
  };
  
  return { user, loading, loadUser, updateUser };
}
```

**Characteristics**:
- Pure functions hoặc hooks
- Framework-agnostic (có thể test outside React)
- Reusable across multiple UI components

---

#### Layer 3: Data Access Layer (Infrastructure Layer)

**Responsibility**:
- HTTP requests (fetch, axios)
- GraphQL queries/mutations
- WebSocket connections
- LocalStorage/IndexedDB access
- Cache management
- Error handling (network errors, retries)
- Request/response transformation

**Không làm**:
- Business logic (calculations, validations)
- UI concerns

**Code trong layer này**:
```
- API clients
- GraphQL hooks (Apollo, React Query)
- Repository pattern implementations
- HTTP interceptors
- Cache policies
```

**Example**:
```javascript
// userAPI.ts - Data Access Layer
const API_BASE = 'https://api.example.com';

export const userAPI = {
  async fetchUser(userId) {
    const response = await fetch(`${API_BASE}/users/${userId}`);
    if (!response.ok) throw new Error('Failed to fetch user');
    return response.json();
  },
  
  async updateUser(userId, updates) {
    const response = await fetch(`${API_BASE}/users/${userId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(updates)
    });
    if (!response.ok) throw new Error('Failed to update user');
    return response.json();
  },
  
  async fetchUserOrders(userId) {
    const response = await fetch(`${API_BASE}/users/${userId}/orders`);
    if (!response.ok) throw new Error('Failed to fetch orders');
    return response.json();
  }
};

// With React Query - also Data Access Layer
export function useUserQuery(userId) {
  return useQuery(['user', userId], () => userAPI.fetchUser(userId));
}
```

**Characteristics**:
- Không biết về business rules
- Chỉ lo communication với external systems
- Handle network errors, retries, caching

---

### Dependency Flow (Critical!)

```
UI Layer
  ↓ depends on
Business Logic Layer  
  ↓ depends on
Data Access Layer
  ↓ depends on
External APIs/DB
```

**Rule**: 
- **Top layers** depend on **bottom layers** ✅
- **Bottom layers** NEVER depend on **top layers** ❌

**Example**:
```javascript
// ✅ GOOD: UI calls Business Logic
function UserProfile() {
  const { user, updateUser } = useUserProfile(userId); // Business Logic
  return <div>...</div>;
}

// ❌ BAD: Business Logic imports UI components
function useUserProfile() {
  return <UserAvatar />; // NO! Business logic shouldn't know about UI
}
```

---

### File Structure Example

```
src/
├── ui/                          # Presentation Layer
│   ├── components/
│   │   ├── UserProfile.tsx
│   │   ├── UserAvatar.tsx
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── Settings.tsx
│   ├── hooks/                   # UI hooks
│   │   ├── useToggle.ts
│   │   ├── useMediaQuery.ts
│
├── domain/                      # Business Logic Layer
│   ├── user/
│   │   ├── userService.ts       # Business rules
│   │   ├── userValidation.ts
│   │   ├── useUserProfile.ts    # Business hooks
│   ├── order/
│   │   ├── orderService.ts
│   │   ├── orderCalculations.ts
│   ├── shared/
│   │   ├── validators.ts
│   │   ├── formatters.ts
│
├── infrastructure/              # Data Access Layer
│   ├── api/
│   │   ├── userAPI.ts
│   │   ├── orderAPI.ts
│   │   ├── httpClient.ts
│   ├── storage/
│   │   ├── localStorage.ts
│   ├── cache/
│   │   ├── cacheManager.ts
```

---

### Benefits của Layered Architecture

#### 1. **Separation of Concerns**
```
UI team: Focus on UX, không cần biết API details
Backend team: Change API, chỉ sửa Data Layer
Business analysts: Review business logic layer (readable code)
```

#### 2. **Testability**
```javascript
// Test Business Logic without UI
test('calculateDiscount for premium user', () => {
  const user = { isPremium: true };
  expect(calculateUserDiscount(user)).toBe(0.2);
});

// Test UI without real API
test('UserProfile renders correctly', () => {
  const mockUser = { name: 'John' };
  render(<UserProfile user={mockUser} />);
  expect(screen.getByText('John')).toBeInTheDocument();
});
```

#### 3. **Reusability**
```
Business logic không tied to UI:
- Có thể dùng trong mobile app (React Native)
- Có thể dùng trong CLI tool
- Có thể dùng trong background worker

Data Access Layer có thể swap:
- fetch → axios
- REST → GraphQL
- HTTP → WebSocket
```

#### 4. **Maintainability**
```
Bug trong validation? → Check Business Logic Layer
UI không render đúng? → Check Presentation Layer
API response changed? → Check Data Access Layer

Clear ownership, dễ debug
```

---

### Drawbacks

#### 1. **Boilerplate**
```
Feature đơn giản cần 3 files:
- UserProfile.tsx (UI)
- userService.ts (Business Logic)  
- userAPI.ts (Data Access)

Có thể overkill cho small apps
```

#### 2. **Indirection**
```
Trace code flow phức tạp:
Component → Hook → Service → API → Network

Nhiều abstraction layers → harder to follow
```

#### 3. **Over-engineering**
```
Startup với 5 screens không cần strict layering
Technical debt nếu team không maintain discipline
```

---

### Khi nào dùng Layered Architecture?

#### ✅ Use when:
```
1. Medium-large apps (20+ screens)
2. Multiple developers (need clear boundaries)
3. Complex business logic (e-commerce, fintech)
4. Need to swap infrastructure (REST → GraphQL)
5. Long-term maintenance (5+ years)
6. Testability is priority
```

#### ❌ Don't use when:
```
1. Small apps (<10 screens)
2. Solo developer (overhead not worth it)
3. Prototyping/MVP (speed > structure)
4. Simple CRUD (no complex business logic)
5. Team unfamiliar (learning curve)
```

---

## 2. Feature-Based Structure - Organize by Business Feature

### Concept

**Core idea**: Group code theo **business features/domains**, không phải technical layers.

```
Thay vì:
components/
services/
apis/

Làm:
features/
  users/
  orders/
  products/
```

**Philosophy**: Code liên quan nhau nên ở gần nhau.

---

### Structure Example

```
src/
├── features/
│   ├── users/
│   │   ├── components/
│   │   │   ├── UserProfile.tsx
│   │   │   ├── UserAvatar.tsx
│   │   │   ├── UserList.tsx
│   │   ├── hooks/
│   │   │   ├── useUserProfile.ts
│   │   │   ├── useUserList.ts
│   │   ├── api/
│   │   │   ├── userAPI.ts
│   │   ├── services/
│   │   │   ├── userService.ts
│   │   ├── types/
│   │   │   ├── User.ts
│   │   ├── utils/
│   │   │   ├── userValidation.ts
│   │   └── index.ts              # Public API
│   │
│   ├── orders/
│   │   ├── components/
│   │   │   ├── OrderList.tsx
│   │   │   ├── OrderDetail.tsx
│   │   ├── hooks/
│   │   │   ├── useOrders.ts
│   │   ├── api/
│   │   │   ├── orderAPI.ts
│   │   ├── services/
│   │   │   ├── orderCalculations.ts
│   │   └── index.ts
│   │
│   ├── products/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── index.ts
│
├── shared/                       # Cross-feature code
│   ├── components/               # Generic UI components
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   ├── hooks/
│   │   ├── useToggle.ts
│   ├── utils/
│   │   ├── formatters.ts
│   ├── types/
│   │   ├── common.ts
│
├── pages/                        # Route pages (orchestrate features)
│   ├── Dashboard.tsx
│   ├── UserProfilePage.tsx
│   ├── OrdersPage.tsx
```

---

### Key Principles

#### 1. **Feature isolation**
```
Mỗi feature là mini-app:
- Có components riêng
- Có logic riêng
- Có API riêng
- Có types riêng

Can develop independently
```

#### 2. **Public API (index.ts)**
```javascript
// features/users/index.ts
export { UserProfile, UserList } from './components';
export { useUserProfile, useUserList } from './hooks';
export type { User, UserRole } from './types';

// Other features import từ public API:
import { UserProfile } from '@/features/users';

// ❌ KHÔNG import directly:
import { UserProfile } from '@/features/users/components/UserProfile';
```

**Why**: Encapsulation, dễ refactor internal structure

#### 3. **Shared folder cho cross-cutting concerns**
```
Button, Input → Dùng ở nhiều features → shared/
formatDate, validateEmail → Dùng ở nhiều features → shared/
```

#### 4. **Pages orchestrate features**
```javascript
// pages/Dashboard.tsx
import { UserProfile } from '@/features/users';
import { OrderList } from '@/features/orders';
import { ProductGrid } from '@/features/products';

function Dashboard() {
  return (
    <Layout>
      <UserProfile userId={currentUser.id} />
      <OrderList userId={currentUser.id} />
      <ProductGrid featured={true} />
    </Layout>
  );
}
```

**Pages không có logic**, chỉ compose features.

---

### Benefits

#### 1. **Colocation** (Code gần nhau)
```
Work on User feature:
- Mở folder users/
- Tất cả code liên quan ở đây
- Không phải jump giữa components/, services/, api/

Faster development, easier navigation
```

#### 2. **Scalability**
```
Add new feature = Add new folder
Delete feature = Delete folder

Clear boundaries, parallel development
```

#### 3. **Team ownership**
```
Team A owns users/ feature
Team B owns orders/ feature

Merge conflicts ít hơn, clear responsibility
```

#### 4. **Code splitting natural**
```javascript
// Lazy load features
const Users = lazy(() => import('@/features/users'));
const Orders = lazy(() => import('@/features/orders'));

// Each feature = 1 chunk
```

#### 5. **Testability**
```
Test users/ feature in isolation
Mock dependencies at feature boundary
```

---

### Drawbacks

#### 1. **Shared code challenges**
```
Problem: UserAvatar dùng ở users/ và orders/
Solution 1: Move to shared/ (nhưng mất colocation)
Solution 2: Duplicate (nhưng không DRY)
Solution 3: Create ui-components/ package (monorepo)
```

#### 2. **Feature dependencies**
```
Problem: orders/ feature cần User type từ users/

❌ BAD: orders/ import từ users/
→ Circular dependency risk

✅ GOOD: Shared types trong shared/types/
```

#### 3. **Over-fragmentation**
```
Too many small features:
- auth/
- login/
- logout/
- password-reset/

Better: Group related vào auth/
```

#### 4. **Hard to refactor across features**
```
Rename User → Customer impacts nhiều features
Need IDE refactoring tools
```

---

### Khi nào dùng Feature-Based?

#### ✅ Use when:
```
1. Multiple teams (feature ownership)
2. Large apps (50+ screens)
3. Microservices backend (features map to services)
4. Frequent feature additions/deletions
5. Need code splitting by feature
6. Domain-driven development
```

#### ❌ Don't use when:
```
1. Small apps (<20 screens)
2. Lots of shared UI components (colocation breaks down)
3. Features heavily interdependent
4. Team unfamiliar (learning curve)
```

---

## 3. Domain-Driven Design (DDD) - For Large, Complex Apps

### Concept

**Origin**: Eric Evans (2003) - Book "Domain-Driven Design"

**Core idea**: Structure code around **business domains**, not technical concerns.

**Philosophy**: 
```
Software reflects real-world business domains
Developers speak same language as business stakeholders
Code organized like business thinks about problems
```

---

### Key DDD Concepts

#### 1. **Domain** - Business area
```
E-commerce domains:
- Catalog (products, categories)
- Shopping Cart
- Order Management
- Payment Processing
- User Management
- Shipping & Fulfillment
```

Mỗi domain có:
- Business rules riêng
- Data models riêng
- API endpoints riêng

#### 2. **Bounded Context** - Logical boundary
```
Trong domain lớn, chia nhỏ thành contexts:

Order Management domain:
├── Order Creation (bounded context 1)
├── Order Fulfillment (bounded context 2)
└── Order Returns (bounded context 3)

Each context có models riêng:
- Order trong Creation context ≠ Order trong Fulfillment context
```

**Why**: Tránh "God object" (Order object 100 properties)

#### 3. **Entities** - Objects với identity
```
Entity = Object có unique ID, lifecycle, state changes

Examples:
- User (id, name, email) - có thể update
- Order (id, items, status) - có thể fulfill, cancel
- Product (id, name, price) - có thể update price

Key: ID unique, có thể track qua time
```

#### 4. **Value Objects** - Immutable data
```
Value Object = No identity, defined by values

Examples:
- Address (street, city, zip)
- Money (amount, currency)
- DateRange (start, end)

Key: 2 objects equal nếu values giống nhau
```

#### 5. **Aggregates** - Consistency boundary
```
Aggregate = Cluster entities + value objects

Example: Order Aggregate
├── Order (root entity)
├── OrderItems (entities)
└── ShippingAddress (value object)

Rules:
- External code chỉ reference Order root
- Changes through Order (not directly to OrderItems)
- Transaction boundary (all or nothing)
```

#### 6. **Repositories** - Data access abstraction
```
Repository = Interface to fetch/save aggregates

interface OrderRepository {
  findById(id: string): Promise<Order>;
  save(order: Order): Promise<void>;
  findByUser(userId: string): Promise<Order[]>;
}

Hide infrastructure details (SQL, MongoDB, API)
```

#### 7. **Domain Services** - Business logic không thuộc entity
```
When logic không belong to 1 entity:

calculateShippingCost(order, address, warehouse)
→ Logic involves Order + Address + Warehouse
→ Không nên trong Order entity alone

→ Create ShippingService
```

---

### DDD Structure in React

```
src/
├── domains/
│   ├── catalog/                 # Bounded Context 1
│   │   ├── entities/
│   │   │   ├── Product.ts
│   │   │   ├── Category.ts
│   │   ├── value-objects/
│   │   │   ├── Money.ts
│   │   │   ├── ProductImage.ts
│   │   ├── repositories/
│   │   │   ├── ProductRepository.ts
│   │   │   ├── CategoryRepository.ts
│   │   ├── services/
│   │   │   ├── ProductSearchService.ts
│   │   │   ├── PricingService.ts
│   │   ├── components/          # UI for this domain
│   │   │   ├── ProductCard.tsx
│   │   │   ├── ProductList.tsx
│   │   ├── hooks/
│   │   │   ├── useProducts.ts
│   │   └── index.ts
│   │
│   ├── cart/                    # Bounded Context 2
│   │   ├── entities/
│   │   │   ├── Cart.ts
│   │   │   ├── CartItem.ts
│   │   ├── repositories/
│   │   │   ├── CartRepository.ts
│   │   ├── services/
│   │   │   ├── CartService.ts
│   │   │   ├── DiscountService.ts
│   │   ├── components/
│   │   │   ├── CartView.tsx
│   │   │   ├── CartSummary.tsx
│   │   └── index.ts
│   │
│   ├── orders/                  # Bounded Context 3
│   │   ├── aggregates/
│   │   │   ├── Order/
│   │   │   │   ├── Order.ts         # Aggregate root
│   │   │   │   ├── OrderItem.ts
│   │   │   │   ├── ShippingAddress.ts
│   │   ├── repositories/
│   │   │   ├── OrderRepository.ts
│   │   ├── services/
│   │   │   ├── OrderService.ts
│   │   │   ├── OrderFulfillmentService.ts
│   │   ├── components/
│   │   └── index.ts
│   │
│   ├── users/                   # Bounded Context 4
│   │   ├── entities/
│   │   │   ├── User.ts
│   │   ├── value-objects/
│   │   │   ├── Email.ts
│   │   │   ├── UserRole.ts
│   │   ├── repositories/
│   │   │   ├── UserRepository.ts
│   │   ├── services/
│   │   │   ├── AuthService.ts
│   │   │   ├── UserProfileService.ts
│   │   ├── components/
│   │   └── index.ts
│
├── shared/                      # Shared kernel
│   ├── domain/                  # Cross-domain concepts
│   │   ├── value-objects/
│   │   │   ├── Money.ts
│   │   │   ├── Address.ts
│   │   ├── interfaces/
│   │   │   ├── Repository.ts
│   ├── infrastructure/          # Technical concerns
│   │   ├── api/
│   │   ├── storage/
│   ├── ui/                      # Generic UI
│   │   ├── components/
│
├── application/                 # Application services (use cases)
│   ├── usecases/
│   │   ├── PlaceOrderUseCase.ts
│   │   ├── CheckoutUseCase.ts
│
├── pages/                       # Presentation (orchestration)
│   ├── ProductPage.tsx
│   ├── CartPage.tsx
│   ├── CheckoutPage.tsx
```

---

### DDD Example: Order Domain

```typescript
// domains/orders/entities/Order.ts (Entity)
export class Order {
  constructor(
    public readonly id: OrderId,      // Value Object
    public userId: UserId,            // Value Object
    private items: OrderItem[],       // Entities
    private status: OrderStatus,      // Enum
    private shippingAddress: Address  // Value Object
  ) {}
  
  // Business logic trong entity
  addItem(product: Product, quantity: number) {
    if (this.status !== OrderStatus.Draft) {
      throw new Error('Cannot modify submitted order');
    }
    
    const existingItem = this.items.find(i => i.productId.equals(product.id));
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this.items.push(new OrderItem(product, quantity));
    }
  }
  
  calculateTotal(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.calculateSubtotal()),
      Money.zero()
    );
  }
  
  submit() {
    if (this.items.length === 0) {
      throw new Error('Cannot submit empty order');
    }
    this.status = OrderStatus.Submitted;
  }
  
  // Aggregate root controls access
  getItems(): readonly OrderItem[] {
    return Object.freeze([...this.items]);
  }
}

// domains/orders/value-objects/OrderId.ts (Value Object)
export class OrderId {
  constructor(private readonly value: string) {
    if (!value) throw new Error('OrderId cannot be empty');
  }
  
  equals(other: OrderId): boolean {
    return this.value === other.value;
  }
  
  toString(): string {
    return this.value;
  }
}

// domains/orders/repositories/OrderRepository.ts (Repository Interface)
export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
  findByUser(userId: UserId): Promise<Order[]>;
}

// infrastructure/repositories/HttpOrderRepository.ts (Implementation)
export class HttpOrderRepository implements OrderRepository {
  async findById(id: OrderId): Promise<Order | null> {
    const response = await fetch(`/api/orders/${id.toString()}`);
    const data = await response.json();
    return this.mapToEntity(data);
  }
  
  async save(order: Order): Promise<void> {
    await fetch(`/api/orders/${order.id.toString()}`, {
      method: 'PUT',
      body: JSON.stringify(this.mapToDTO(order))
    });
  }
  
  // Mapping logic hidden trong repository
  private mapToEntity(data: any): Order { /* ... */ }
  private mapToDTO(order: Order): any { /* ... */ }
}

// application/usecases/PlaceOrderUseCase.ts (Application Service)
export class PlaceOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private paymentService: PaymentService,
    private emailService: EmailService
  ) {}
  
  async execute(orderId: OrderId): Promise<void> {
    // 1. Load aggregate
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new Error('Order not found');
    
    // 2. Business logic
    order.submit();
    
    // 3. Process payment (domain service)
    await this.paymentService.charge(order.calculateTotal());
    
    // 4. Save aggregate
    await this.orderRepo.save(order);
    
    // 5. Side effects
    await this.emailService.sendOrderConfirmation(order);
  }
}

// React component using DDD
function CheckoutPage() {
  const placeOrder = usePlaceOrder(); // Hook wraps Use Case
  
  const handleCheckout = async () => {
    await placeOrder.execute(new OrderId(currentOrderId));
  };
  
  return <button onClick={handleCheckout}>Place Order</button>;
}
```

---

### Benefits của DDD

#### 1. **Ubiquitous Language**
```
Developers, business, stakeholders dùng cùng terms:
- "Order" không phải "Request" hay "Transaction"
- "Fulfill" không phải "Process" hay "Complete"
- "Aggregate" = term cả team hiểu

Code reflects business conversations
```

#### 2. **Business logic encapsulation**
```
Business rules trong domain entities:
- Cannot add item to submitted Order
- Cannot apply discount > 50%
- Must have valid shipping address

Logic không scattered trong UI/API
```

#### 3. **Testability**
```javascript
// Test business logic without UI/API
test('Order cannot be submitted when empty', () => {
  const order = new Order(/* ... */);
  expect(() => order.submit()).toThrow('Cannot submit empty order');
});

// No mocking needed, pure domain logic
```

#### 4. **Flexibility**
```
Swap infrastructure:
- SQLOrderRepository → MongoOrderRepository
- Domain logic unchanged

Swap UI framework:
- React → Vue → Angular
- Domain logic unchanged
```

#### 5. **Scalability**
```
Clear bounded contexts:
- Separate teams per context
- Independent deployment (microservices)
- Minimal coupling
```

---

### Drawbacks

#### 1. **Complexity**
```
Overhead:
- Entities, Value Objects, Repositories, Services
- Many abstraction layers
- Lots of files

Overkill cho simple CRUD apps
```

#### 2. **Learning curve**
```
Team cần học:
- DDD concepts (aggregates, bounded contexts)
- Design patterns
- Domain modeling skills

3-6 months ramp-up time
```

#### 3. **Over-engineering risk**
```
Developers create abstractions unnecessarily:
- UserNameValueObject cho simple string
- Every entity có repository dù chỉ đọc

Premature optimization
```

#### 4. **Frontend-backend misalignment**
```
Frontend DDD models ≠ Backend API models
Need mapping layer (tedious)

Example:
- Backend: Order có 50 fields
- Frontend: Chỉ cần 10 fields
- Must maintain mapping code
```

---

### Khi nào dùng DDD?

#### ✅ Use when:
```
1. Large, complex business domains (fintech, healthcare, logistics)
2. Many business rules (discounts, eligibility, workflows)
3. Long-term projects (5-10+ years)
4. Multiple teams
5. Need clear business-dev communication
6. Microservices architecture
7. Domain experts available (business stakeholders)
```

#### ❌ Don't use when:
```
1. Simple CRUD apps
2. Prototyping/MVP (overkill)
3. Small team (<5 devs)
4. Short-term project (<1 year)
5. No domain complexity (basic blog, landing page)
6. Team unfamiliar with DDD (steep learning curve)
```

---

## So sánh 3 Architectures

```markdown

| Aspect | Layered | Feature-Based | DDD |
|--------|---------|---------------|-----|
| **Organize by** | Technical concern | Business feature | Business domain |
| **Best for** | Medium apps | Large apps | Complex domains |
| **Team size** | 3-10 devs | 5-20 devs | 10+ devs |
| **Complexity** | Medium | Low-Medium | High |
| **Learning curve** | Low | Low | High |
| **Coupling** | Low (clear layers) | Medium (feature deps) | Very low (bounded contexts) |
| **Scalability** | Good | Excellent | Excellent |
| **Business alignment** | Low | Medium | High |
| **Testability** | Good | Good | Excellent |
| **Boilerplate** | Medium | Low | High |
| **Example apps** | E-commerce, dashboards | SaaS platforms | Banking, logistics, healthcare |
```

Đây nhé bạn! Copy được rồi 👍

---

## Decision Framework

### Small App (<20 screens, 1-3 devs)
```
Recommendation: Simple folder structure
src/
├── components/
├── pages/
├── hooks/
├── utils/

No fancy architecture needed
```

### Medium App (20-50 screens, 3-10 devs)
```
Option A: Layered Architecture
- Clear separation
- Easy to learn
- Good for teams familiar with MVC

Option B: Feature-Based
- Better colocation
- Faster development
- Good for parallel teams
```

### Large App (50+ screens, 10+ devs)
```
Option A: Feature-Based + Layered (Hybrid)
features/
  users/
    ui/
    domain/
    infrastructure/

Option B: DDD (if complex business logic)
domains/
  orders/
    entities/
    services/
    repositories/
```

### Very Large (100+ screens, multiple teams, microservices)
```
Recommendation: DDD with Bounded Contexts
- Map to microservices
- Clear domain boundaries
- Team ownership per context
```

---

## Key Takeaways

### Layered Architecture
- **When**: Need clear technical separation
- **Benefit**: Testable, reusable, maintainable
- **Drawback**: Boilerplate, indirection
- **Use**: Medium apps, teams familiar with MVC

### Feature-Based
- **When**: Multiple features, parallel development
- **Benefit**: Colocation, scalability, team ownership
- **Drawback**: Shared code challenges
- **Use**: Large apps, feature teams

### DDD
- **When**: Complex business logic, long-term projects
- **Benefit**: Business alignment, flexibility, testability
- **Drawback**: Complexity, learning curve, over-engineering risk
- **Use**: Enterprise apps, complex domains

---

**Final advice**: 
> Start simple, evolve as needed. Most apps don't need DDD. Feature-based structure is sweet spot for 80% of React apps. Only go DDD if domain complexity justifies it.

Bạn có câu hỏi gì về các architectures này không? Hoặc muốn discuss real-world scenarios? 🤔