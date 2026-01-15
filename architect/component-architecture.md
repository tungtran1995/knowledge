# Component Architecture 

## 1. Atomic Design - Phân cấp component theo kích thước

### Concept

**Nguồn gốc**: Brad Frost (2013) - Lấy cảm hứng từ hóa học (atoms → molecules → organisms)

**Core idea**: UI là tập hợp các thành phần nhỏ kết hợp thành thành phần lớn hơn.

```
Atoms (nhỏ nhất, không chia được)
  ↓
Molecules (nhóm atoms có ý nghĩa)
  ↓
Organisms (sections phức tạp)
  ↓
Templates (layouts)
  ↓
Pages (instances cụ thể)
```

---

### 1. Atoms - Không thể chia nhỏ hơn

**Definition**: HTML elements cơ bản hoặc components UI nguyên tử.

**Đặc điểm**:
- Không thể breakdown thành components nhỏ hơn
- Highly reusable
- Generic, không biết context business
- Style-focused

**Ví dụ thực tế**:
```
- Button
- Input
- Label
- Icon
- Avatar
- Badge
- Spinner
```

**Khi nào là Atom?**
- Tự nó đứng một mình có ý nghĩa visual
- Không chứa business logic
- Có thể dùng ở bất kỳ đâu

**Anti-pattern**: Atom biết về domain business
```
❌ BAD: UserAvatar (biết về User entity)
✅ GOOD: Avatar (generic, nhận props image/name)
```

---

### 2. Molecules - Nhóm atoms có mục đích

**Definition**: Kết hợp nhiều atoms thành functional unit đơn giản.

**Đặc điểm**:
- Có 1 single purpose rõ ràng
- Kết hợp 2-5 atoms
- Có một chút logic (validation, formatting)
- Reusable nhưng có context

**Ví dụ**:
```
SearchBar = Input + Button + Icon
FormField = Label + Input + ErrorMessage
UserCard = Avatar + Text + Badge
PriceTag = Icon + Text
```

**Khi nào là Molecule?**
- Atoms alone không đủ để express intent
- Có relationship giữa các atoms (Label cho Input)
- Still generic enough để reuse

**Critical point**: Molecule vẫn không biết data từ đâu
```
SearchBar không biết gọi API nào
FormField không biết validate rules cụ thể
```

---

### 3. Organisms - Sections phức tạp, có business logic

**Definition**: Distinct section của interface với business context.

**Đặc điểm**:
- Kết hợp molecules + atoms
- Có business logic
- Biết về domain (users, products, orders)
- Có thể fetch data, handle state
- Less reusable (domain-specific)

**Ví dụ**:
```
Header = Logo + Navigation + SearchBar + UserMenu
ProductList = nhiều ProductCard + Pagination + Filters
CommentSection = CommentForm + CommentList
ShoppingCart = CartItems + PriceCalculator + CheckoutButton
```

**Khi nào là Organism?**
- Cần biết business context
- Có state management phức tạp
- Có side effects (API calls)
- Specific đến 1 domain area

---

### 4. Templates - Page layouts (without data)

**Definition**: Wireframes, grid systems, page structure.

**Đặc điểm**:
- Arrange organisms vào layout
- Define responsive breakpoints
- No real data (placeholders)
- Focus on structure, không phải content

**Ví dụ**:
```
DashboardTemplate = Header + Sidebar + MainContent + Footer
ArticleTemplate = Header + Breadcrumbs + Article + Sidebar + Comments
ProfileTemplate = Header + ProfileHero + Tabs + Content
```

**Tại sao cần Templates?**
- Design system showcase (Storybook)
- Test layouts với mock data
- Consistent page structure
- Easier responsive design

---

### 5. Pages - Templates + real data

**Definition**: Templates filled với actual data.

**Đặc điểm**:
- Specific instances (Home page, Product #123)
- Real API calls
- Routing
- Page-level state

**Ví dụ**:
```
HomePage = DashboardTemplate với actual user data
ProductDetailPage = ProductTemplate với product ID từ URL
```

---

## Tại sao dùng Atomic Design?

### ✅ Benefits

#### 1. **Consistency**
```
Reuse atoms → Consistent look & feel
Mọi button giống nhau
Mọi input giống nhau
Design system tự nhiên emerge
```

#### 2. **Scalability**
```
Thêm feature mới:
- Reuse atoms/molecules có sẵn
- Compose thành organisms mới
- Fast development

Không phải viết lại từ đầu
```

#### 3. **Testability**
```
Test atoms isolation → Dễ test
Test molecules với mock atoms
Test organisms với mock molecules

Bottom-up testing strategy
```

#### 4. **Design-Dev collaboration**
```
Designers:
- Design atoms trong Figma
- Compose molecules
- Build organisms

Developers:
- Implement 1-1 mapping
- Same vocabulary
- Less miscommunication
```

#### 5. **Storybook integration**
```
Showcase atoms individually
Show all button variants
Test molecules in isolation
Document usage
```

---

### ❌ Drawbacks & When NOT to use

#### 1. **Over-engineering small apps**
```
Problem: 5-page website không cần 5-layer hierarchy
Solution: Just use Components + Pages
```

#### 2. **Ambiguous boundaries**
```
Problem: "Is this a large molecule or small organism?"
Reality: Không có câu trả lời đúng
Solution: Team agreement > strict rules
```

#### 3. **File organization nightmare**
```
Problem: Tìm component khó vì nested folders
components/
  atoms/
    button/
      index.tsx
      styles.css
      tests.tsx
  molecules/
    ...

Solution: Flat structure hoặc feature-based
```

#### 4. **Refactoring overhead**
```
Problem: Upgrade molecule → organism requires moving files
Reality: Boundaries change during development
Solution: Don't be too strict
```

---

## Thực tế Production: Simplified Atomic

Nhiều teams không dùng đầy đủ 5 levels, mà simplify thành:

### Variant 1: 3-tier (Most common)
```
Primitives (atoms)
  ↓
Components (molecules)
  ↓
Features (organisms)
```

### Variant 2: 2-tier (Startups)
```
UI Components (generic)
  ↓
Domain Components (business-specific)
```

---

## 2. Presentational vs Container (Legacy Pattern)

### Concept

**Nguồn gốc**: Dan Abramov (2015) - Redux era pattern

**Core idea**: Tách UI rendering khỏi data fetching/state management.

```
Container (Smart)
- Fetch data
- Manage state
- Handle logic
- Pass props to Presentational

Presentational (Dumb)
- Receive props
- Render UI
- No state (or minimal local UI state)
- Pure, predictable
```

---

### Presentational Components

**Characteristics**:
- Nhận props
- Render JSX
- Có thể có local UI state (open/closed, hover)
- Không gọi API
- Không connect Redux/Context

**Example mental model**:
```
Props in → JSX out
Pure function of props
Như <div> nhưng custom
```

**Use cases**:
- Design system components
- Reusable UI elements
- Components dùng nhiều nơi

---

### Container Components

**Characteristics**:
- Fetch data (API calls, GraphQL)
- Connect to state management (Redux, Context)
- Handle side effects
- Minimal UI (thường chỉ wrap Presentational)
- Business logic

**Example mental model**:
```
Data orchestrator
Connect data sources → Presentational components
```

---

### Tại sao pattern này "legacy"?

#### Before (2015-2018): Manual separation
```javascript
// Container
class UserListContainer extends Component {
  state = { users: [] };
  
  componentDidMount() {
    fetch('/api/users')
      .then(res => res.json())
      .then(users => this.setState({ users }));
  }
  
  render() {
    return <UserList users={this.state.users} />;
  }
}

// Presentational
const UserList = ({ users }) => (
  <ul>
    {users.map(user => <li>{user.name}</li>)}
  </ul>
);
```

**Problems**:
- Boilerplate
- 2 files cho 1 feature
- Unclear boundary

#### After (2019+): Hooks blur the line
```javascript
// Modern: Single component với hooks
function UserList() {
  const [users, setUsers] = useState([]);
  
  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(setUsers);
  }, []);
  
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

**Why better?**
- Less boilerplate
- Colocation (logic + UI)
- Hooks are reusable (extract to custom hook if needed)

---

### Modern equivalent: Custom Hooks

**Instead of Container/Presentational split**:
```javascript
// Custom hook (replaces Container logic)
function useUsers() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    setLoading(true);
    fetch('/api/users')
      .then(res => res.json())
      .then(users => {
        setUsers(users);
        setLoading(false);
      });
  }, []);
  
  return { users, loading };
}

// Component (like Presentational, but uses hook)
function UserList() {
  const { users, loading } = useUsers();
  
  if (loading) return <Spinner />;
  
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

**Benefits**:
- Logic reusable (khác components có thể dùng `useUsers()`)
- Single file (easier navigation)
- Still testable (mock hook)

---

### Khi nào vẫn dùng Container pattern?

#### Scenario 1: Reusable presentation layer
```
Có 1 table component dùng nhiều nơi với data khác nhau

DataTable (Presentational) - reusable
  ↑ props
UsersTableContainer - fetch users
ProductsTableContainer - fetch products
OrdersTableContainer - fetch orders
```

#### Scenario 2: Server components (React Server Components)
```
Server Component (Container-like)
- Fetch data on server
- Pass props to Client Component

Client Component (Presentational-like)
- Interactive UI
- Receive props from server
```

---

## 3. Smart vs Dumb Components

### Concept

**Same as Presentational/Container** nhưng khác terminology:
- **Smart** = Container = Has logic
- **Dumb** = Presentational = Just UI

**Tại sao terminology này better?**
- Dễ hiểu: Smart (biết nhiều) vs Dumb (biết ít)
- Không imply physical separation (Container vs Presentational suggests 2 files)

---

### Smart Components

**Definition**: Components biết về business, state, side effects.

**Responsibilities**:
- Data fetching
- State management
- Event handling logic
- Routing
- Authentication checks

**Example scenarios**:
```
Dashboard - knows current user, fetches metrics
ShoppingCart - knows items, calculates total, checkout logic
ChatRoom - WebSocket connection, messages state
```

**Mental model**: Brain của feature

---

### Dumb Components

**Definition**: Components chỉ render UI, không biết data source.

**Responsibilities**:
- Render props
- Local UI state (hover, focus)
- CSS animations
- Accessibility

**Example scenarios**:
```
Button - just renders, doesn't know what happens onClick
Card - displays info, doesn't know where info comes from
Modal - shows/hides, doesn't know content
```

**Mental model**: Puppet được Smart component điều khiển

---

### Spectrum thinking (Modern approach)

**Reality**: Không phải binary (Smart/Dumb), mà là spectrum:

```
Pure Dumb ←----------------→ Pure Smart
         ↑      ↑       ↑
      Button  FormField  Dashboard
```

**Most components ở giữa**: Có một chút logic nhưng không quá smart.

**Example**: Form field
```javascript
function FormField({ label, value, onChange, validate }) {
  const [error, setError] = useState(null);
  
  const handleBlur = () => {
    const err = validate(value);
    setError(err);
  };
  
  return (
    <div>
      <label>{label}</label>
      <input 
        value={value} 
        onChange={onChange}
        onBlur={handleBlur}
      />
      {error && <span>{error}</span>}
    </div>
  );
}
```

**Is this Smart or Dumb?**
- Has state (error) → Smart-ish
- Doesn't fetch data → Dumb-ish
- Answer: **Somewhere in between** ✅

---

### Modern guideline: Don't overthink it

```
Đừng stress về Smart/Dumb classification
Focus on:
1. Is it testable?
2. Is it reusable?
3. Is it readable?

Nếu 3 câu trả lời YES → Good component, regardless of label
```

---

## 4. Compound Components - Components làm việc cùng nhau

### Concept

**Core idea**: Một nhóm components được design để work together, share internal state.

**Analogy**: `<select>` và `<option>` trong HTML
```html
<select>
  <option value="1">One</option>
  <option value="2">Two</option>
</select>
```
- `<select>` manage state (selected value)
- `<option>` render options và communicate với `<select>`
- Không thể dùng `<option>` mà không có `<select>`

---

### Tại sao cần Compound Components?

#### Problem: Too many props (API hell)
```javascript
// ❌ BAD: Prop hell
<Accordion
  items={[...]}
  renderItem={(item) => ...}
  renderHeader={(item) => ...}
  renderContent={(item) => ...}
  onExpand={(item) => ...}
  defaultExpanded={0}
  allowMultiple={false}
  animated={true}
  // 20+ props...
/>
```

**Problems**:
- API phức tạp
- Không flexible (hard to customize)
- Props drilling

#### Solution: Compound components
```javascript
// ✅ GOOD: Compound pattern
<Accordion>
  <Accordion.Item>
    <Accordion.Header>Section 1</Accordion.Header>
    <Accordion.Content>Content here</Accordion.Content>
  </Accordion.Item>
  
  <Accordion.Item>
    <Accordion.Header>Section 2</Accordion.Header>
    <Accordion.Content>More content</Accordion.Content>
  </Accordion.Item>
</Accordion>
```

**Benefits**:
- Declarative, dễ đọc
- Flexible layout (thứ tự, nesting)
- Ít props hơn
- Shared state (Accordion knows which Item expanded)

---

### How it works: Context API

**Mechanism**: Parent component provide state via Context, children consume.

**Conceptual structure**:
```
Accordion (Provider)
  ├─ State: expandedIndex
  ├─ Functions: toggle(index)
  └─ Context.Provider
      ↓
  Accordion.Item (Consumer)
      ├─ Reads: isExpanded from Context
      ├─ Calls: toggle() on click
      └─ Renders based on state
```

---

### Real-world examples

#### 1. **Tabs**
```javascript
<Tabs>
  <Tabs.List>
    <Tabs.Tab>Profile</Tabs.Tab>
    <Tabs.Tab>Settings</Tabs.Tab>
  </Tabs.List>
  
  <Tabs.Panel>Profile content</Tabs.Panel>
  <Tabs.Panel>Settings content</Tabs.Panel>
</Tabs>
```

**Shared state**: `activeTab` index

#### 2. **Modal/Dialog**
```javascript
<Modal>
  <Modal.Trigger>
    <button>Open</button>
  </Modal.Trigger>
  
  <Modal.Content>
    <Modal.Header>Title</Modal.Header>
    <Modal.Body>Content</Modal.Body>
    <Modal.Footer>
      <Modal.Close>Cancel</Modal.Close>
    </Modal.Footer>
  </Modal.Content>
</Modal>
```

**Shared state**: `isOpen` boolean

#### 3. **Dropdown Menu**
```javascript
<Menu>
  <Menu.Button>Actions</Menu.Button>
  <Menu.Items>
    <Menu.Item onClick={handleEdit}>Edit</Menu.Item>
    <Menu.Item onClick={handleDelete}>Delete</Menu.Item>
  </Menu.Items>
</Menu>
```

**Shared state**: `isOpen`, keyboard navigation, focus management

---

### Benefits của Compound Components

#### 1. **Flexibility**
```
User có thể arrange sub-components tùy ý
Thêm/bớt elements dễ dàng
Custom styling mỗi part riêng
```

#### 2. **Separation of Concerns**
```
Mỗi sub-component có responsibility riêng
Header lo header
Content lo content
Trigger lo trigger
```

#### 3. **Implicit Communication**
```
Không cần prop drilling
Sub-components tự communicate qua Context
Clean API
```

#### 4. **Discoverable API**
```
IDE autocomplete: Accordion. → shows .Item, .Header, .Content
Clear structure
Self-documenting
```

---

### Drawbacks

#### 1. **Must use together**
```
❌ Cannot use Accordion.Item outside Accordion
❌ Must follow structure

Workaround: Provide helpful error messages
```

#### 2. **Context overhead**
```
Every re-render của parent → re-render children
Mitigation: Optimize Context (split contexts, memoization)
```

#### 3. **Learning curve**
```
Users phải hiểu relationship giữa components
Documentation cần rõ ràng
```

---

### Khi nào dùng Compound Components?

#### ✅ Use when:
```
1. Component có nhiều "parts" có state chung
   (Accordion, Tabs, Menu, Modal)

2. Muốn flexible layout
   (User sắp xếp parts tùy thích)

3. API quá nhiều props
   (>10-15 props → consider Compound)

4. Sub-components không meaningful alone
   (Tabs.Panel vô nghĩa nếu không có Tabs)
```

#### ❌ Don't use when:
```
1. Component đơn giản (Button, Input)
   (Không cần sub-components)

2. Sub-components independent
   (Có thể dùng riêng lẻ)

3. Performance critical
   (Context re-renders có thể slow)

4. Team không familiar
   (Learning curve)
```

---

## Tổng kết: Khi nào dùng pattern nào?

### Decision Matrix

| Pattern | Team Size | App Complexity | Use Case | Modern Alternative |
|---------|-----------|----------------|----------|-------------------|
| **Atomic Design** | Medium-Large (5+ devs) | High | Design system, component library | Feature-based structure |
| **Container/Presentational** | Any | Medium | Redux era | Custom Hooks |
| **Smart/Dumb** | Any | Any | Mental model, not strict files | Spectrum thinking |
| **Compound** | Any | High UI components | Complex stateful UI (Tabs, Accordion) | No alternative |

---

### Practical Recommendations

#### Small App (1-3 devs, <20 screens)
```
Structure:
- components/ (UI components)
- pages/ (routes)
- hooks/ (custom hooks)

Pattern: Không cần strict patterns, focus on delivery
```

#### Medium App (3-10 devs, 20-100 screens)
```
Structure:
- components/
  - ui/ (atoms/molecules equivalent)
  - features/ (organisms equivalent)
- pages/
- hooks/

Pattern: 
- Informal Atomic (ui vs features)
- Custom hooks thay Container
- Compound cho complex UI
```

#### Large App (10+ devs, 100+ screens, design system)
```
Structure:
- packages/
  - design-system/ (Atomic Design)
  - features/ (Smart components)
  - shared/ (utilities)

Pattern:
- Full Atomic Design
- Compound components library
- Dedicated design system team
```

---

## Key Takeaways

### Atomic Design
- **Purpose**: Organize components by complexity
- **Benefit**: Consistency, reusability, scalability
- **Drawback**: Over-engineering small apps, ambiguous boundaries
- **Use**: Design systems, large teams

### Container/Presentational (Legacy)
- **Purpose**: Separate data logic from UI
- **Benefit**: Clear separation, testability
- **Drawback**: Boilerplate, 2 files per feature
- **Modern**: Custom Hooks handle logic, single component

### Smart/Dumb
- **Purpose**: Same as Container/Presentational, better naming
- **Reality**: Spectrum, not binary
- **Guideline**: Don't overthink, focus on testable + reusable

### Compound Components
- **Purpose**: Group related components with shared state
- **Benefit**: Flexible, declarative API, less props
- **Use**: Tabs, Accordion, Menu, Modal
- **When**: Complex UI with parts needing coordination

---

**Final advice**: 
> Patterns là tools, không phải rules. Chọn based on team size, complexity, và deadline. Consistency trong team > perfect pattern.
