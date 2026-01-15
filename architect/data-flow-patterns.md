# Data Flow Patterns - Góc nhìn Principal

Được, tôi sẽ tiếp tục với cùng format và độ sâu như document về Component Architecture.

---

## 1. Unidirectional Data Flow - Data chỉ chảy một chiều

### Concept

**Nguồn gốc**: React (2013) - Phản ứng với two-way binding của Angular 1.x

**Core idea**: Data chảy theo một hướng duy nhất, từ parent xuống child, không có reverse flow tự động.

```
State (Single Source of Truth)
  ↓
View = f(State)
  ↓
User Interaction
  ↓
Action/Event
  ↓
State Update
  ↓ (loop back)
View Re-render
```

---

### Bản chất: Predictability > Flexibility

**Triết lý**:
- State là immutable snapshot tại một thời điểm
- View chỉ là hàm thuần túy của state: `UI = render(state)`
- Mọi thay đổi phải qua explicit action
- Time travel debugging possible (vì biết chính xác state tại mỗi thời điểm)

**Mental model**:
```
Giống dòng nước chảy xuống thác:
- Chỉ chảy từ trên xuống dưới
- Muốn nước lên lại phải bơm (action)
- Có thể "tua lại" dòng chảy (time-travel)
```

---

### So sánh: Unidirectional vs Bidirectional

#### Bidirectional (Two-way binding)
```
Ví dụ: Angular 1.x ng-model

View ←→ Model

<input ng-model="user.name" />

User type → Model tự update
Model change → View tự update

Problems:
1. Không biết ai trigger change
2. Cascade updates (A→B→C→A infinite loop)
3. Debug nightmare: "Tại sao biến này lại đổi?"
4. Performance: Digest cycle check toàn bộ watchers
```

#### Unidirectional (One-way binding)
```
Ví dụ: React

View ← State
  ↓
Action → State Update

<input 
  value={user.name} 
  onChange={(e) => updateUser({name: e.target.value})} 
/>

User type → trigger onChange → explicit update
State change → View re-render

Benefits:
1. Biết chính xác flow
2. No cascade (chỉ top-down)
3. Dễ debug: Trace từ action → reducer → state
4. Predictable: Cùng state = cùng UI
```

---

### Data flow cycle chi tiết

#### Step 1: Initial State
```
Application khởi tạo với initial state
const initialState = {
  users: [],
  loading: false,
  error: null
};
```

#### Step 2: Render
```
View = function(state)
Component nhận state qua props/context
Render UI based on state
```

#### Step 3: User Interaction
```
User click button
User type input
User scroll
→ Trigger event handler
```

#### Step 4: Dispatch Action
```
Event handler dispatch action
Action = plain object mô tả "what happened"

{
  type: 'FETCH_USERS_REQUEST',
  payload: { page: 1 }
}
```

#### Step 5: State Update
```
Reducer/setState nhận action
Tạo NEW state (immutable)
oldState + action = newState
```

#### Step 6: Re-render
```
State changed → notify subscribers
Components re-render với new state
Cycle lặp lại
```

---

### Tại sao Unidirectional quan trọng?

#### 1. **Debugging**
```
Bidirectional:
"Biến này đổi rồi, không biết ai đổi?"
→ Search toàn project
→ Tìm 10 chỗ có thể đổi
→ Set breakpoint khắp nơi

Unidirectional:
"Biến này đổi"
→ Check Redux DevTools
→ Thấy action "UPDATE_USER" triggered lúc 10:30:05
→ Click vào action → xem stack trace
→ Fix trong 2 phút
```

#### 2. **Testability**
```
Bidirectional:
- Phải test integration (View ↔ Model sync)
- Mock complicated
- Flaky tests (timing issues)

Unidirectional:
- Test reducer pure function: input → output
- Test component: props → render
- Test action creators: args → action object
- Easy to mock, deterministic
```

#### 3. **Mental Model**
```
Bidirectional:
- Phải track 2 directions
- "Nếu A đổi thì B đổi, B đổi thì C đổi, C đổi lại A..."
- Cannot reason about code

Unidirectional:
- Chỉ theo 1 chiều
- Action → Reducer → State → View
- Hình dung được flow trong đầu
```

#### 4. **Team Scalability**
```
1 dev: Bidirectional OK (nhớ hết flow)
5 devs: Bắt đầu rối (ai đổi gì?)
10 devs: Chaos (merge conflicts, bugs)

Unidirectional:
- Clear contract: Action interface
- Mỗi dev làm 1 feature = 1 action
- Ít conflict
- Code review dễ hơn
```

#### 5. **Time-travel Debugging**
```
Redux DevTools có thể:
- "Jump" đến bất kỳ state nào trong history
- Replay actions
- Skip actions
- Export/import state

Chỉ possible vì:
- State immutable (history preserved)
- Actions explicit (biết từng bước)
```

---

### Implementation patterns

#### Pattern 1: React useState (Simple)
```javascript
Component có local state
setState trigger re-render
Flow: User action → setState → re-render

Use case: UI state (modal open/close, form input)
```

#### Pattern 2: Lifting State Up (Medium)
```javascript
State nằm ở parent component
Children nhận props + callback
Callback gọi parent's setState

Use case: Shared state giữa siblings
```

#### Pattern 3: Context API (Medium-Large)
```javascript
Provider component hold state
Consumer components subscribe
Flow: Provider setState → context notify → consumers re-render

Use case: Global-ish state (theme, auth, i18n)
```

#### Pattern 4: Redux/Zustand (Large)
```javascript
Centralized store
Components dispatch actions
Reducers update store
Store notify → components re-render

Use case: Complex state, many components share
```

---

### Trade-offs cần biết

#### ❌ Drawbacks

**1. Boilerplate**
```
Bidirectional:
<input ng-model="name" />  // Done

Unidirectional:
const [name, setName] = useState('');
<input 
  value={name} 
  onChange={(e) => setName(e.target.value)} 
/>
// More code
```

**2. Performance concern (if naive)**
```
Every state change → full subtree re-render
Solution: React.memo, useMemo, useCallback
```

**3. Learning curve**
```
Junior devs quen imperative (jQuery):
$('#name').val('John'); // Direct

React:
setName('John'); // Indirect, need understand cycle
```

**4. Verbose for simple cases**
```
Simple form input requires:
- State variable
- Event handler
- Value prop
- onChange prop

Bidirectional: 1 line
```

#### ✅ Benefits

**1. Predictable**
```
Same input (state) = Same output (UI)
No surprises
```

**2. Debuggable**
```
Clear action history
Easy to trace
```

**3. Testable**
```
Pure functions
Isolated testing
```

**4. Scalable**
```
Works with large teams
Clear contracts
```

---

### Khi nào dùng?

#### ✅ Use Unidirectional when:
```
1. Team > 3 people
2. App > 10 screens
3. Shared state giữa nhiều components
4. Cần debug complex interactions
5. Long-term maintenance (>6 months)
```

#### ❌ Don't overthink when:
```
1. Prototype/MVP
2. Landing page (mostly static)
3. Internal tool (1 dev, simple CRUD)
4. Deadline tight

→ Use simple useState, move to Redux later if needed
```

---

## 2. Flux Architecture - Cụ thể hóa Unidirectional Flow

### Concept

**Nguồn gốc**: Facebook (2014) - Giải quyết pain points của MVC trong large-scale app

**Core idea**: Flux không phải library, là một **architecture pattern** với 4 thành phần rõ ràng.

```
Action → Dispatcher → Store → View
           ↑                      ↓
           └──────────────────────┘
```

---

### 4 thành phần chính

#### 1. **Action** - "Cái gì đã xảy ra"

**Definition**: Plain object mô tả event.

**Structure**:
```javascript
{
  type: 'USER_LOGGED_IN',    // Required: action type
  payload: {                 // Optional: data
    userId: 123,
    userName: 'Alice'
  }
}
```

**Characteristics**:
- Immutable
- Serializable (có thể JSON.stringify)
- Descriptive name (past tense: LOGGED_IN, not LOGIN)
- No logic inside

**Mental model**: Newspaper headline
```
"USER_LOGGED_IN" = "User đã login"
"ITEM_ADDED_TO_CART" = "Item đã được thêm vào cart"
```

**Anti-patterns**:
```javascript
❌ BAD: Action có logic
{
  type: 'UPDATE_USER',
  payload: {
    user: fetchUserFromAPI() // NO! Side effect
  }
}

❌ BAD: Generic action
{
  type: 'UPDATE',  // Update cái gì?
  data: {...}
}

✅ GOOD: Specific, declarative
{
  type: 'USER_PROFILE_UPDATE_SUCCEEDED',
  payload: { userId: 123, name: 'Alice' }
}
```

---

#### 2. **Dispatcher** - "Central hub"

**Definition**: Pub/sub mechanism, route actions đến stores.

**Responsibilities**:
- Receive actions
- Broadcast đến TẤT CẢ stores (not selective)
- Đảm bảo actions xử lý **tuần tự** (no parallel)

**Characteristics**:
- Singleton (only 1 dispatcher per app)
- No business logic
- Simple event bus

**Mental model**: Switchboard operator cũ
```
Action gọi đến → Dispatcher answer → Broadcast đến tất cả stores
```

**Key insight: Sequential processing**
```
Action A dispatched
  → Store 1 process A
  → Store 2 process A
  → Store 3 process A
  → Done

Then Action B dispatched
  → Store 1 process B
  → ...
  
NO parallel processing
Prevents race conditions
```

**waitFor() - Dependency management**:
```javascript
// UserStore cần chờ AuthStore xử lý trước
dispatcher.register((action) => {
  if (action.type === 'USER_LOGIN') {
    dispatcher.waitFor([AuthStore.dispatchToken]);
    // AuthStore đã xử lý xong
    // Giờ mới xử lý logic của UserStore
  }
});
```

---

#### 3. **Store** - "State container + Logic"

**Definition**: Hold application state + business logic.

**Responsibilities**:
- Store state
- Handle actions (via callback registered với Dispatcher)
- Emit change events
- Provide getters (NO setters)

**Characteristics**:
- Multiple stores (UserStore, CartStore, ProductStore)
- Stores **độc lập**, không gọi nhau trực tiếp
- State private (only mutate internally)

**Mental model**: Database + Business logic layer
```
Store = Model (MVC) nhưng có logic
```

**Key difference vs Redux**:
```
Flux: Multiple stores
- UserStore
- CartStore  
- ProductStore
Each store handle own domain

Redux: Single store
- { users: {...}, cart: {...}, products: {...} }
All in one object
```

**Store structure pattern**:
```javascript
class UserStore extends EventEmitter {
  constructor() {
    this._users = [];
    
    // Register với dispatcher
    this.dispatchToken = dispatcher.register((action) => {
      switch (action.type) {
        case 'USER_FETCHED':
          this._users = action.payload.users;
          this.emit('change');
          break;
      }
    });
  }
  
  // Public getter (NO setter)
  getUsers() {
    return this._users;
  }
  
  // Private mutation
  _addUser(user) {
    this._users.push(user);
  }
}
```

**Anti-patterns**:
```javascript
❌ BAD: Store gọi Store khác
class CartStore {
  addItem(item) {
    const user = UserStore.getUser(); // Direct dependency
    // ...
  }
}

✅ GOOD: Stores độc lập, communicate qua actions
class CartStore {
  constructor() {
    dispatcher.register((action) => {
      if (action.type === 'USER_LOGGED_OUT') {
        this._clearCart(); // React to action
      }
    });
  }
}
```

---

#### 4. **View** - "React Components"

**Definition**: UI layer, display state from stores.

**Responsibilities**:
- Subscribe to stores
- Render UI based on store state
- Dispatch actions on user interaction

**Characteristics**:
- Can be nested (components tree)
- Controller-Views (top-level) vs child views
- Re-render when store emits change

**Mental model**: Observer pattern
```
View "watch" Store
Store change → View notified → View re-render
```

**Controller-View pattern**:
```javascript
// Top-level component
class UserListContainer extends Component {
  constructor() {
    this.state = {
      users: UserStore.getUsers()
    };
  }
  
  componentDidMount() {
    // Subscribe to store
    UserStore.on('change', this._onChange);
  }
  
  componentWillUnmount() {
    UserStore.removeListener('change', this._onChange);
  }
  
  _onChange = () => {
    this.setState({
      users: UserStore.getUsers()
    });
  };
  
  render() {
    return <UserList users={this.state.users} />;
  }
}
```

---

### Complete Flux Flow - Chi tiết từng bước

#### Scenario: User click "Add to Cart" button

**Step 1: User Interaction**
```javascript
// View
<button onClick={() => {
  CartActions.addItem(product);
}}>
  Add to Cart
</button>
```

**Step 2: Action Creator**
```javascript
// CartActions.js
const CartActions = {
  addItem(product) {
    dispatcher.dispatch({
      type: 'ADD_TO_CART',
      payload: { product }
    });
  }
};
```

**Step 3: Dispatcher Broadcast**
```javascript
// Dispatcher send action đến ALL stores
dispatcher.dispatch(action);
// → UserStore nhận action (ignore vì không liên quan)
// → CartStore nhận action (handle)
// → ProductStore nhận action (ignore)
```

**Step 4: Store Handle Action**
```javascript
// CartStore.js
class CartStore extends EventEmitter {
  constructor() {
    this._items = [];
    
    dispatcher.register((action) => {
      switch (action.type) {
        case 'ADD_TO_CART':
          this._items.push(action.payload.product);
          this.emit('change'); // Notify views
          break;
      }
    });
  }
  
  getItems() {
    return this._items;
  }
}
```

**Step 5: View Update**
```javascript
// CartView.js
class CartView extends Component {
  componentDidMount() {
    CartStore.on('change', this._onChange);
  }
  
  _onChange = () => {
    this.setState({
      items: CartStore.getItems()
    });
  };
  
  render() {
    return (
      <div>
        Cart: {this.state.items.length} items
      </div>
    );
  }
}
```

---

### Tại sao Flux giải quyết pain points của MVC?

#### Problem 1: Cascade Updates (MVC)
```
MVC:
Controller update Model A
  → Model A notify View X
    → View X update Model B
      → Model B notify View Y
        → View Y update Model C
          → ...

= Infinite loop potential
= Cannot predict order
```

**Flux solution**:
```
Action dispatched
  → ALL stores update synchronously
  → Emit change once
  → Views re-render
  → Done

= One-way flow
= Predictable order
```

#### Problem 2: Shared State (MVC)
```
MVC:
Multiple controllers modify same model
Model notify all views
Views out of sync

Example:
UserController.updateName()
ProfileController.updateName()
→ Race condition
```

**Flux solution**:
```
Only stores can mutate state
External code dispatch actions
Actions sequential (no race)

Example:
dispatch({ type: 'UPDATE_NAME', payload: {name} })
→ UserStore handle
→ Atomic update
```

#### Problem 3: Complex Dependencies (MVC)
```
MVC:
Model A depends on Model B
Model B depends on Model C
→ Hard to reason about order
```

**Flux solution**:
```
Stores độc lập
If need dependency: waitFor()

CartStore.waitFor([UserStore.dispatchToken])
→ Explicit dependency
→ Clear order
```

---

### Flux vs Redux - Key Differences

| Aspect | Flux | Redux |
|--------|------|-------|
| **Stores** | Multiple stores (UserStore, CartStore) | Single store ({ users, cart }) |
| **Dispatcher** | Explicit Dispatcher singleton | No dispatcher (reducers) |
| **State mutation** | Stores tự mutate internal state | Reducers return new state (pure) |
| **Store logic** | Stores có methods, inheritance | Reducers là pure functions |
| **Registration** | `dispatcher.register(callback)` | No registration (auto) |

**Redux simplification**:
```
Flux:
- Multiple stores (complex)
- Dispatcher (extra concept)
- Stores mutable internally

Redux:
- Single store (simpler mental model)
- No dispatcher (reducers compose)
- Immutable updates (easier to debug)
```

**Why Redux won?**
```
1. Simpler API (less boilerplate)
2. Time-travel debugging (immutability)
3. Middleware ecosystem (thunks, sagas)
4. Better devtools
5. Single store = easier serialization
```

---

### Khi nào dùng Flux (pure) today?

**Reality**: Almost never.

**Redux/Zustand replaced Flux** vì:
- Less boilerplate
- Better tooling
- Larger community

**Only use pure Flux if**:
```
1. Legacy codebase đã dùng Flux
2. Academic purposes (học concept)
3. Custom implementation cần waitFor() (rare)
```

**Modern equivalent**:
```
Flux principles still valid:
- Unidirectional flow ✓
- Actions as events ✓
- Centralized state ✓

Implementation: Redux, Zustand, Jotai
```

---

## 3. CQRS - Command Query Responsibility Segregation

### Concept

**Nguồn gốc**: Greg Young (2010) - From Domain-Driven Design

**Core idea**: Tách biệt **write operations** (Commands) và **read operations** (Queries) sử dụng models khác nhau.

```
Traditional:
Same model cho read và write
User Model: getUser(), updateUser()

CQRS:
Write Model (Commands): updateUser(), deleteUser()
Read Model (Queries): getUserById(), getUserList()
→ 2 models riêng biệt, optimized cho từng use case
```

---

### Bản chất: Separation of Concerns

**Mental model**: Restaurant
```
Traditional:
1 person nhận order, nấu, serve
→ Slow, inefficient

CQRS:
Waiter nhận order (Command)
Chef nấu (Process)
Different waiter serve (Query)
→ Fast, specialized
```

**Triết lý**:
```
Reads và Writes có requirements khác nhau:
- Reads: Fast, denormalized, cached
- Writes: Validated, normalized, transactional

→ Optimize riêng
```

---

### Thành phần chính

#### 1. **Commands** - "Thay đổi state"

**Definition**: Operations that mutate state.

**Characteristics**:
- Imperative (CreateUser, UpdateProduct, DeleteOrder)
- May fail (validation)
- Side effects (write to DB, send email)
- Return success/error (not data)

**Example commands**:
```javascript
class CreateUserCommand {
  constructor(name, email, password) {
    this.name = name;
    this.email = email;
    this.password = password;
  }
}

class UpdateProductPriceCommand {
  constructor(productId, newPrice) {
    this.productId = productId;
    this.newPrice = newPrice;
  }
}
```

**Command Handler**:
```javascript
class CreateUserCommandHandler {
  async handle(command) {
    // Validation
    if (!isValidEmail(command.email)) {
      throw new Error('Invalid email');
    }
    
    // Business logic
    const hashedPassword = await hash(command.password);
    
    // Write to DB
    await db.users.insert({
      name: command.name,
      email: command.email,
      password: hashedPassword
    });
    
    // Side effects
    await emailService.sendWelcome(command.email);
    
    // Return void or success
    return { success: true };
  }
}
```

---

#### 2. **Queries** - "Đọc data"

**Definition**: Operations that fetch data, không mutate.

**Characteristics**:
- Declarative (GetUserById, ListProducts, SearchOrders)
- Always succeed (hoặc return empty)
- No side effects
- Return data (not success/error)

**Example queries**:
```javascript
class GetUserByIdQuery {
  constructor(userId) {
    this.userId = userId;
  }
}

class ListProductsQuery {
  constructor(page, pageSize, filters) {
    this.page = page;
    this.pageSize = pageSize;
    this.filters = filters;
  }
}
```

**Query Handler**:
```javascript
class GetUserByIdQueryHandler {
  async handle(query) {
    // Direct read, no validation
    const user = await db.users.findById(query.userId);
    
    // Có thể denormalize, join data
    const userWithOrders = {
      ...user,
      orders: await db.orders.findByUserId(user.id),
      stats: await db.stats.getForUser(user.id)
    };
    
    return userWithOrders;
  }
}
```

---

### CQRS Architecture

#### Simple CQRS (Single DB)
```
             Commands                    Queries
                ↓                          ↓
         Command Handler             Query Handler
                ↓                          ↓
         Write to DB  ←─────────→    Read from DB
                         (same DB)
```

**Use case**: Separation of concerns, not scale.

---

#### Advanced CQRS (Separate DBs)
```
Commands                          Queries
   ↓                                 ↓
Command Handler               Query Handler
   ↓                                 ↓
Write DB (normalized)         Read DB (denormalized)
   ↓                                 ↑
   └──────── Sync ──────────────────┘
           (Event/Queue)
```

**Key insight**: Write DB ≠ Read DB
```
Write DB:
- Relational (PostgreSQL)
- Normalized (3NF)
- Transactional

Read DB:
- NoSQL (MongoDB, Elasticsearch)
- Denormalized (embedded docs)
- Optimized for queries
```

---

### Event Sourcing (thường đi kèm CQRS)

**Concept**: Thay vì lưu current state, lưu **stream of events**.

```
Traditional:
User { id: 1, name: 'Alice', email: 'alice@example.com' }

Event Sourcing:
Event 1: UserCreated { name: 'Alice', email: 'alice@ex.com' }
Event 2: UserEmailUpdated { email: 'alice@example.com' }
Event 3: UserNameUpdated { name: 'Alice Smith' }

Current state = replay events
```

**Flow với CQRS + Event Sourcing**:
```
1. Command dispatched
2. Command handler validate
3. Emit events (UserCreated, OrderPlaced)
4. Events stored in Event Store
5. Event handler update Read DB
6. Query read from Read DB
```

**Benefits**:
```
- Audit trail (biết chính xác history)
- Time travel (replay đến bất kỳ thời điểm nào)
- Projections (build multiple read models từ same events)
```

---

### Tại sao cần CQRS?

#### Problem 1: Read/Write requirements khác nhau

**Scenario**: E-commerce product page
```
Read (Product Detail Page):
- Show product info
- Show reviews (avg rating, count)
- Show related products
- Show inventory
→ Need JOIN 4-5 tables
→ Slow if normalized

Write (Add Review):
- Insert 1 row vào Reviews table
- Fast
```

**Without CQRS**:
```
Same model handle both
→ Reads slow (many JOINs)
→ Writes complicated (update denormalized data)
```

**With CQRS**:
```
Write Model: Normalized
- Products table
- Reviews table
- Inventory table

Read Model: Denormalized
- ProductDetails document {
    product: {...},
    reviews: {...},
    relatedProducts: [...],
    inventory: {...}
  }

Write fast (1 table)
Read fast (1 document)
```

---

#### Problem 2: Scale differently

**Reality**:
```
Most apps: Reads >> Writes (90% reads, 10% writes)
```

**Without CQRS**:
```
Same DB cho reads và writes
→ Scale DB = scale cả 2 (lãng phí)
```

**With CQRS**:
```
Write DB: 1 instance (enough)
Read DB: 10 replicas (cache, CDN)

Cost efficient
```

---

#### Problem 3: Complex business logic vs Simple queries

**Scenario**: Order processing
```
Write (Place Order):
- Validate user
- Check inventory
- Calculate price (discounts, taxes)
- Reserve items
- Process payment
- Send confirmation
→ Complex, transactional

Read (Order History):
- SELECT * FROM orders WHERE userId = ?
→ Simple
```

**Without CQRS**:
```
Order Model có cả 2:
- placeOrder() - complex
- getOrders() - simple

→ Mixed responsibilities
```

**With CQRS**:
```
Command: PlaceOrderCommand + Handler (complex logic)
Query: GetOrdersQuery + Handler (simple read)

→ Clear separation
```

---

### Implementation trong Frontend

#### Pattern 1: API separation
```javascript
// Commands API
const commandAPI = {
  createUser: (data) => POST('/api/commands/users', data),
  updateUser: (id, data) => PUT(`/api/commands/users/${id}`, data),
  deleteUser: (id) => DELETE(`/api/commands/users/${id}`)
};

// Queries API
const queryAPI = {
  getUser: (id) => GET(`/api/queries/users/${id}`),
  listUsers: (filters) => GET('/api/queries/users', filters),
  searchUsers: (q) => GET('/api/queries/users/search', { q })
};
```

**Benefits**:
```
- Different base URLs (scale independently)
- Different caching strategies (queries cached, commands not)
- Clear intent
```

---

#### Pattern 2: State Management (Redux example)
```javascript
// Commands (actions)
const actions = {
  createUser: (data) => ({
    type: 'CREATE_USER_COMMAND',
    payload: data
  }),
  updateUser: (id, data) => ({
    type: 'UPDATE_USER_COMMAND',
    payload: { id, data }
  })
};

// Queries (selectors)
const selectors = {
  getUser: (state, id) => state.users.byId[id],
  listUsers: (state) => Object.values(state.users.byId),
  searchUsers: (state, query) => 
    Object.values(state.users.byId).filter(u => 
      u.name.includes(query)
    )
};
```

**Separation**:
```
Actions → mutate state
Selectors → read state (memoized với reselect)
```

---

#### Pattern 3: React Query / SWR
```javascript
// Commands (mutations)
const useCreateUser = () => {
  return useMutation({
    mutationFn: (data) => api.createUser(data),
    onSuccess: () => {
      queryClient.invalidateQueries(['users']);
    }
  });
};

// Queries
const useUser = (id) => {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => api.getUser(id),
    staleTime: 60000, // Cache 1 min
  });
};
```

**CQRS principles**:
```
Mutations: invalidate cache, no return data
Queries: cached, optimized, return data
```

---

### Trade-offs

#### ❌ Drawbacks

**1. Complexity**
```
CQRS adds layers:
- Command/Query split
- Handlers
- Sync logic (if separate DBs)

Small app: Overkill
```

**2. Eventual Consistency**
```
Write DB → Event → Read DB (async)
→ User update name, refresh, see old name
→ Confusing UX

Mitigation: Optimistic UI updates
```

**3. Data duplication**
```
Same data in Write DB và Read DB
→ Storage cost
→ Sync logic complexity
```

**4. Learning curve**
```
Team phải hiểu:
- Commands vs Queries
- Event sourcing (if used)
- Eventual consistency

→ Training cost
```

---

#### ✅ Benefits

**1. Scalability**
```
Scale reads và writes independently
Read replicas, caching
```

**2. Performance**
```
Optimized models:
- Write: Fast inserts
- Read: Fast queries (denormalized)
```

**3. Flexibility**
```
Multiple read models từ same writes:
- SQL cho reports
- Elasticsearch cho search
- Redis cho real-time

Build thêm projections không impact writes
```

**4. Security**
```
Commands: Strict validation, auth

Queries: Less strict (readonly)

Different permissions
```

---

### Khi nào dùng CQRS?

#### ✅ Use when:
```
1. Reads và Writes có requirements rất khác
   (Complex writes, simple reads)

2. Read/Write ratio rất skewed
   (90% reads → scale reads only)

3. Multiple representations của same data
   (SQL reports, NoSQL search, Redis cache)

4. Complex domain logic
   (Banking, E-commerce với nhiều rules)

5. Need audit trail
   (Healthcare, Finance)
```

#### ❌ Don't use when:
```
1. Simple CRUD app
   (Blog, TODO list)

2. Small team (<5 devs)
   (Overhead không worth)

3. Tight deadlines
   (CQRS adds complexity)

4. Reads và Writes similar complexity
   (No benefit)
```

---

### CQRS trong React ecosystem

**Simple CQRS (không separate DBs)**:
```
Use case: Large React app, complex forms

Strategy:
- React Query mutations = Commands
- React Query queries = Queries
- Same API, khác endpoints
- Clear separation of concerns
```

**Advanced CQRS (với GraphQL)**:
```
Commands: GraphQL mutations
Queries: GraphQL queries (với @defer, @stream)

Server có separate resolvers:
- Mutation resolvers → write logic
- Query resolvers → optimized reads
```

---

## Tổng kết: 3 Patterns So sánh

| Pattern | Complexity | Use Case | Team Size | Learning Curve |
|---------|------------|----------|-----------|----------------|
| **Unidirectional Flow** | Low | All apps | Any | Easy |
| **Flux** | Medium | Legacy only (use Redux thay thế) | Medium | Medium |
| **CQRS** | High | Complex domains, high scale | Large (10+) | Hard |

---

### Decision Tree

```
Need state management?
├─ YES → Use Unidirectional Flow
│   ├─ Simple: React useState/useReducer
│   ├─ Medium: Context API
│   └─ Complex: Redux/Zustand
│
└─ Have complex domain logic?
    ├─ YES → Consider CQRS
    │   ├─ High read/write ratio? → CQRS với separate DBs
    │   └─ Complex validations? → CQRS commands pattern
    │
    └─ NO → Stick với simple Unidirectional
```

---

### Practical Recommendations

#### Small App (1-3 devs)
```
✅ Unidirectional với useState/Context
❌ Flux (legacy)
❌ CQRS (overkill)
```

#### Medium App (3-10 devs)
```
✅ Unidirectional với Redux/Zustand
✅ Simple CQRS (mutations vs queries API split)
❌ Full CQRS với separate DBs
```

#### Large App (10+ devs)
```
✅ Unidirectional với Redux + middleware
✅ CQRS commands/queries pattern
✅ Event sourcing (if need audit trail)
✅ Separate read/write DBs (if scale needed)
```

---

## Key Takeaways

### Unidirectional Data Flow
- **Core**: Data chỉ chảy một chiều, predictable
- **Benefit**: Debuggable, testable, scalable với teams
- **Drawback**: Boilerplate, learning curve
- **Use**: Mọi app > toy projects

### Flux
- **Core**: 4 components (Action, Dispatcher, Store, View)
- **Historical**: Flux principles good, implementation replaced by Redux
- **Modern**: Học concept, dùng Redux/Zustand

### CQRS
- **Core**: Tách Commands (write) và Queries (read)
- **Benefit**: Scale independently, optimized models
- **Drawback**: Complexity, eventual consistency
- **Use**: Complex domains, high scale, audit trail needed

---

**Final Advice**:
> Start simple (Unidirectional với useState/Redux). Add CQRS chỉ khi **pain points rõ ràng** (slow queries, complex writes, scale issues). Don't premature optimize.