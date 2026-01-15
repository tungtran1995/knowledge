# JavaScript Core Concepts

## 1. Execution Context & Scope

### **Execution Context là gì?**

**Execution Context** = Môi trường JavaScript chạy code

**3 loại:**
- **Global EC**: Top-level, chạy đầu tiên, duy nhất
- **Function EC**: Tạo mỗi khi gọi function
- **Eval EC**: Tránh dùng

---

### **Components của Execution Context**

**Mỗi EC chứa:**

**1. Variable Environment**
- Lưu `var` declarations
- Function declarations
- Arguments object (functions)

**2. Lexical Environment**
- Lưu `let/const` declarations
- Outer reference (scope chain)
- `this` binding

**3. this Value**
- Xác định theo binding rules
- Arrow functions không có `this` riêng

---

### **2 Phases: Creation → Execution**

#### **Phase 1: Creation Phase (Setup)**

**Điều xảy ra:**
- Tạo Variable Environment & Lexical Environment
- **Hoisting diễn ra ở đây**
- Xác định `this`
- Setup scope chain

**var hoisting:**
- Binding created
- **Initialized = undefined** ngay lập tức
- Sẵn sàng truy cập (nhưng = undefined)

**let/const hoisting:**
- Binding created
- **NOT initialized** → Ở trạng thái "uninitialized"
- Temporal Dead Zone (TDZ) bắt đầu

**Function declarations:**
- Hoisted toàn bộ (declaration + body)
- Có thể gọi trước khai báo

#### **Phase 2: Execution Phase (Run)**

**Code chạy line-by-line:**
- Assignments thực thi
- Functions được gọi
- TDZ kết thúc khi đến dòng khai báo let/const

---

### **Call Stack - Execution Context Stack**

**Cơ chế:**
- LIFO (Last In, First Out)
- Global EC luôn ở bottom
- Function EC push/pop khi call/return

**Flow:**
```
1. Global EC pushed
2. Call function → Function EC pushed
3. Function returns → Function EC popped
4. Back to previous EC
5. Program ends → Global EC popped
```

---

### **Scope Chain**

**Concept:**
- Chain of Lexical Environments
- Inner scope → Outer scope → Global
- **Lexical (Static)**: Determined tại write-time

**Lookup process:**
1. Tìm trong current Lexical Environment
2. Không thấy → Tìm outer reference
3. Repeat đến Global
4. Không thấy → ReferenceError

**Why "Lexical":**
- Scope xác định bởi **vị trí code được viết**
- Không phụ thuộc nơi function được gọi
- Predictable, compile-time determination

---

## 2. Variable Declarations

### **var vs let vs const - Quick Compare**

| Feature | var | let | const |
|---------|-----|-----|-------|
| **Scope** | Function | Block | Block |
| **Re-declare** | ✅ OK | ❌ Error | ❌ Error |
| **Re-assign** | ✅ OK | ✅ OK | ❌ Error |
| **Hoisting** | undefined | TDZ | TDZ |
| **Global pollution** | ✅ Yes | ❌ No | ❌ No |

---

### **Hoisting Behavior (Execution Context perspective)**

#### **var trong Creation Phase:**

**Setup:**
- Binding created trong Variable Environment
- **Initialized = undefined** automatically
- Access ngay được (nhưng = undefined)

**Why undefined:**
- Legacy design (1995)
- Tránh crashes
- Trade-off: Dễ bugs (use before define)

#### **let/const trong Creation Phase:**

**Setup:**
- Binding created trong Lexical Environment
- **NOT initialized** → "uninitialized" state
- **TDZ starts** từ đây

**TDZ (Temporal Dead Zone):**
- Từ **đầu block** đến **dòng khai báo**
- Access trong TDZ → ReferenceError
- Design: **Fail fast** - catch bugs sớm

**TDZ ends:**
- Khi code chạy đến dòng khai báo
- Variable initialized
- Safe to access

---

### **const Immutability - 3 Levels**

**Level 1: const (Binding only)**
- Cannot reassign binding
- Content vẫn mutable (objects/arrays)

**Level 2: Object.freeze (Shallow)**
- Properties không thay đổi được
- Nested objects vẫn mutable
- Silently fails (strict mode: TypeError)

**Level 3: Deep freeze (True immutability)**
- Recursive freeze
- All nested levels immutable
- Cần custom implementation

**Rule:** const chỉ khóa BINDING, không khóa VALUE

---

### **Block Scope vs Function Scope**

#### **Function Scope (var):**

**Characteristics:**
- Chỉ bị giới hạn bởi function boundaries
- **Bỏ qua blocks**: if, for, while, try/catch
- Khai báo bằng var luôn được hoist lên đầu function scope (chỉ hoist declaration)

**Global scope pollution:**
- var trong global scope → Creates `window` property
- Namespace collision risk
- Third-party scripts có thể access/override

#### **Block Scope (let/const):**

**Characteristics:**
- Bị giới hạn bởi `{}` gần nhất
- Mỗi block = new scope
- Không pollute global object

**Loop behavior đặc biệt:**
- `let` trong loop header → Per-iteration binding
- Mỗi iteration có copy riêng
- Giải quyết closure problem

---

### **Common Pitfalls**

#### **1. Loop Closures với var**

**Problem:**
- var = function scope → Chỉ 1 binding cho toàn loop
- Tất cả closures reference cùng 1 variable
- Khi callbacks chạy, loop đã xong → Final value

**Solution:**
- Dùng let → Per-iteration binding
- Mỗi closure có copy riêng

#### **2. TDZ Edge Cases**

**typeof operator:**
- Undeclared variable: "undefined" (safe)
- TDZ variable: ReferenceError (intentional)
- Design để catch bugs

**Default parameters:**
- Parameters evaluated left-to-right
- TDZ applies: Earlier params cannot reference later ones

**Destructuring:**
- Same TDZ rules
- Right-hand evaluated first

**Sibling scope:**
- Inner declaration shadows outer
- TDZ của inner block chặn access outer variable

#### **3. Function Hoisting**

**Declaration vs Expression:**
- Declaration: Hoisted toàn bộ

```javascript
foo();

function foo() {}
```

- Expression: var foo → hoisted & initialized = undefined, Function expression không hoisted

```javascript
foo(); // ❌ TypeError
var foo = function () {};
```

- Call function expression trước assign → TypeError

---

### **Best Practices**

**Decision flow:**
```
Cần reassign?
├─ No → const ✅
└─ Yes → let ✅

NEVER var ❌
```

**Guidelines:**

**1. Mặc định dùng const**
- Thể hiện rõ ý định: giá trị không bị gán lại
- Tránh vô tình reassignment
- Tăng độ đúng đắn & dễ đọc của code

**2. Chỉ dùng let khi cần gán lại**
- Biến đếm vòng lặp
- Biến tích lũy
- Biến theo dõi trạng thái

**3. Never var**
- Function scope gây hiểu lầm
- Global pollution
- Không có lợi ích so với let/const

**4. Declare at top of scope**
- Hiểu rõ sự khác nhau giữa creation và initialization
- Tránh truy cập biến trước khi được khởi tạo (TDZ – Temporal Dead Zone)
- Ưu tiên scope hẹp

**5. Object.freeze for constants**
- Ngăn việc mutate ngoài ý muốn các thuộc tính cấp cao nhất (top-level)
- Mặc định chỉ shallow freeze
- Chỉ dùng deep freeze khi thật sự cần thiết

---

### **Connection: EC → Hoisting → Scope**

**Mental model:**

```
Execution Context Created
  ↓
Conceptual Creation Phase
  ↓
Variable Environment (var)
  → binding created & initialized = undefined

Lexical Environment (let / const)
  → binding created but uninitialized (TDZ)
  ↓
Execution Phase
  ↓
Code executes line-by-line
  ↓
Initialization happens at declaration
  ↓
TDZ ends
  ↓
Identifier resolution via Scope Chain
  (Current → Outer → Global)

```

**Key insights:**

**1. Hoisting = Creation Phase artifact**
- Hoisting KHÔNG phải là “đưa code lên trên”
- Cấp phát bộ nhớ cho các biến và hàm trước khi chạy

**2. TDZ = Design choice**
- Tránh bug âm thầm (silent bugs) như với var
- Áp dụng nguyên tắc fail fast (sai thì lỗi ngay)
- Ngăn việc dùng biến trước khi khai báo

**3. Scope = Lexical Environment chain**
- Scope là tĩnh (static)
- Được xác định khi viết code, không phải khi chạy

**4. var problems = Function scope + undefined init**
- Global pollution (❌ Không có block scope → if, for vẫn “lọt scope”)
- Boundary của scope không rõ ràng (❌ Tự động gắn vào global object (trong trình duyệt là window))
- Bug khó phát hiện ( ❌ Giá trị undefined che giấu lỗi logic)

**5. let/const benefits = Block scope + TDZ**
- Block scope → ranh giới rõ ràng
- TDZ → phát hiện lỗi sớm
- Không tự động gắn vào global object

---

### **Quick Reference**

| Tình huống | Sử dụng |
|-----------|-----|
| Value never changes | `const` |
| Value changes | `let` |
| Object content mutates | `const` (binding immutable) |
| Object reassigned | `let` |
| Loop counter | `let` |
| Config constants | `const` + `Object.freeze` |

**Core takeaways:**
- Execution Context có 2 phases: Creation (hoisting) → Execution
- var hoisted as undefined, let/const hoisted as uninitialized (TDZ)
- var = function scope + pollution, let/const = block scope + clean
- TDZ catches bugs early via ReferenceError
- Prefer: const > let > never var

---

## 3. this Binding

### **this là gì?**

`this` = Context object mà function được gọi trong đó (runtime binding)

**Không phải**:
- ❌ Function itself
- ❌ Function's scope
- ❌ Where function was defined

**Là**: ✅ **How function was called** (call-site determines this)

---

### **4 Binding Rules (Priority Order)**

#### **Rule 1: new Binding** (Highest Priority)

**Khi dùng `new` operator:**
```javascript
function User(name) {
  this.name = name;
  // Implicit: return this;
}

const user = new User('Alice');
console.log(user.name); // 'Alice'
```

**What happens with `new`:**
1. Tạo empty object mới
2. Link object đến constructor's prototype
3. Bind `this` = new object
4. Execute function body
5. Return `this` (unless explicit return object)

**Use case**: Constructor functions, class constructors

---

#### **Rule 2: Explicit Binding** (call/apply/bind)

**call()**: Invoke immediately
```javascript
function greet(greeting) {
  console.log(`${greeting}, ${this.name}`);
}

const user = { name: 'Bob' };
greet.call(user, 'Hello'); // "Hello, Bob"
```

**apply()**: Same but args as array
```javascript
greet.apply(user, ['Hi']); // "Hi, Bob"
```

**bind()**: Return new function với fixed `this`
```javascript
const greetUser = greet.bind(user);
greetUser('Hey'); // "Hey, Bob"
```

**Hard Binding**: `bind()` cannot be overridden
```javascript
const bound = greet.bind(user);
bound.call(otherUser, 'Hello'); // Still uses 'user', not 'otherUser'
```

**Use cases**:
- Event handlers cần specific context
- Passing methods as callbacks
- Partial application

---

#### **Rule 3: Implicit Binding** (Object method call)

**Rule**: `this` = object chứa method (left of the dot)

```javascript
const person = {
  name: 'Charlie',
  greet() {
    console.log(this.name);
  }
};

person.greet(); // 'Charlie' (this = person)
```

**Implicit Loss - Common Pitfall:**

**Problem 1: Assign to variable**
```javascript
const greet = person.greet;
greet(); // undefined (this = global/undefined in strict)
// Call-site không còn person.greet() → Lost binding
```

**Problem 2: Pass as callback**
```javascript
setTimeout(person.greet, 1000); // undefined
// Callback thực chất là: const fn = person.greet; setTimeout(fn, 1000);
```

**Fix**: Bind explicitly
```javascript
setTimeout(person.greet.bind(person), 1000); // ✅ 'Charlie'
// Or arrow function
setTimeout(() => person.greet(), 1000); // ✅ 'Charlie'
```

---

#### **Rule 4: Default Binding** (Lowest Priority)

**Standalone function invocation:**
```javascript
function foo() {
  console.log(this.name);
}

var name = 'Global'; // var creates global property
foo(); // 'Global' (non-strict) or undefined (strict)
```

**Strict mode:**
```javascript
'use strict';
function foo() {
  console.log(this); // undefined
}
foo();
```

**Use case**: Fallback when no other rule applies (usually unintended)

---

### **Arrow Functions - Lexical this**

**Key difference**: Arrow functions **KHÔNG CÓ `this` riêng**

**Behavior**: Inherit `this` từ enclosing lexical scope (where defined, not called)

```javascript
const obj = {
  name: 'Dave',
  
  regularFunc() {
    console.log(this.name); // 'Dave' (implicit binding)
    
    setTimeout(function() {
      console.log(this.name); // undefined (default binding in callback)
    }, 100);
    
    setTimeout(() => {
      console.log(this.name); // 'Dave' (lexical this từ regularFunc)
    }, 200);
  }
};

obj.regularFunc();
```

**Cannot override arrow this:**
```javascript
const arrow = () => console.log(this);

const obj = { name: 'Test' };
arrow.call(obj); // Ignores call(), uses lexical this
arrow.apply(obj); // Ignores apply()
const bound = arrow.bind(obj); // Binding vô ích
```

**Use cases:**
- ✅ Callbacks cần preserve outer `this`
- ✅ Array methods trong objects
- ❌ Object methods (mất implicit binding)
- ❌ Constructors (SyntaxError)

---

### **Common Pitfalls**

#### **1. Lost Context in Callbacks**
```javascript
class Timer {
  constructor() {
    this.seconds = 0;
  }
  
  start() {
    // ❌ BAD
    setInterval(function() {
      this.seconds++; // this = global/undefined
    }, 1000);
    
    // ✅ GOOD: Arrow function
    setInterval(() => {
      this.seconds++; // Lexical this từ start()
    }, 1000);
    
    // ✅ GOOD: bind
    setInterval(function() {
      this.seconds++;
    }.bind(this), 1000);
  }
}
```

#### **2. Arrow Functions as Methods**
```javascript
const obj = {
  name: 'Eve',
  
  // ❌ BAD: Arrow function as method
  greet: () => {
    console.log(this.name); // undefined (this = global scope)
  },
  
  // ✅ GOOD: Regular method
  greet() {
    console.log(this.name); // 'Eve'
  }
};
```

#### **3. this trong Event Handlers**
```javascript
class Button {
  constructor(element) {
    this.count = 0;
    
    // ❌ BAD: Lost context
    element.addEventListener('click', this.handleClick);
    // this trong handleClick = element, not Button instance
    
    // ✅ GOOD: Bind in constructor
    element.addEventListener('click', this.handleClick.bind(this));
    
    // ✅ BETTER: Arrow function
    element.addEventListener('click', () => this.handleClick());
  }
  
  handleClick() {
    this.count++;
  }
}
```

---

### **Best Practices**

**1. Default to arrow functions cho callbacks:**
```javascript
// ✅ Clean, predictable
fetchData().then(data => this.processData(data));

// vs verbose bind
fetchData().then(function(data) {
  this.processData(data);
}.bind(this));
```

**2. Use regular functions cho object methods:**
```javascript
const api = {
  baseUrl: 'https://api.example.com',
  
  // ✅ Regular function gets implicit binding
  fetch(endpoint) {
    return fetch(`${this.baseUrl}${endpoint}`);
  }
};
```

**3. Bind once trong constructor (React pattern):**
```javascript
class Component {
  constructor() {
    // Bind once, reuse multiple times
    this.handleClick = this.handleClick.bind(this);
  }
  
  handleClick() {
    // this = component instance
  }
}
```

**4. Understand trade-offs:**
- Arrow functions: Cleaner but cannot rebind
- bind(): Explicit but verbose
- call/apply: One-time invocation

---

### **Quick Reference**

| Scenario | this Value |
|----------|------------|
| `new Foo()` | New object created |
| `foo.call(obj)` | `obj` (explicit) |
| `obj.foo()` | `obj` (implicit) |
| `foo()` | global/undefined (default) |
| `() => {}` | Lexical (enclosing scope) |

**Priority**: new > explicit > implicit > default

**Arrow exception**: Ignores all rules, always lexical

---

## 4. CLOSURE

---

```markdown
## 3. Closure - Ứng dụng thực tế

### **Closure là gì? (Định nghĩa chính xác)**

**Closure** = Function + Lexical Environment (biến ngoài scope của nó)

Một function có thể **"nhớ" và truy cập biến của outer scope**, ngay cả khi outer function đã return.

---

### **Cơ chế hoạt động qua Execution Context**

**Điều xảy ra khi function return function:**

**Normally:**
- Function chạy xong → Execution Context bị remove
- Variables → Garbage collected
- Không access được nữa

**Với Closure:**
- Inner function được return → Giữ reference đến outer scope
- JavaScript engine: "Inner cần outer's variables"
- **Không xóa** outer's Lexical Environment
- → Closure hình thành

**Mechanism:**
- Mỗi function có internal [[Scope]] property
- [[Scope]] = Scope Chain từ lúc function **được define**
- Inner function mang theo [[Scope]] → trỏ đến parent's Lexical Environment
- → Variables trong parent environment vẫn reachable → Không bị GC

**Lexical Scoping:**
- "Lexical" = Determined tại write-time, không phải runtime
- Closure capture scope từ **nơi define**, không phải nơi gọi
- Predictable, không bị ảnh hưởng bởi call context

---

### **Tại sao Closure quan trọng?**

**1. Data Privacy**
- Tạo private variables (trước ES2022 # fields)
- Encapsulation không cần classes

**2. Stateful Functions**
- Function "nhớ" state giữa các lần gọi
- Không cần global variables

**3. Function Factories**
- Tạo specialized functions từ generic factory
- Configuration baked-in via closure

**4. Module Pattern (Pre-ES6)**
- Private scope + Public API
- IIFE + Closure = Encapsulation

---

### **Use Cases Thực Tế**

**1. Data Encapsulation:**
Private state không thể access trực tiếp từ bên ngoài, chỉ qua methods.

**2. Event Handlers với State:**
Mỗi handler có private state riêng, không cần global variables.

**3. Debounce/Throttle:**
Closure giữ timer ID, track state giữa các calls.

**4. Memoization:**
Cache results của expensive computations, closure giữ cache.

---

### **Common Pitfalls**

#### **1. Closure trong vòng lặp với var**

**Problem:**
- `var` = function scope → chỉ 1 biến `i`
- Tất cả closures reference cùng 1 biến
- Khi callbacks chạy, loop đã xong → `i` = final value

**Example behavior:**
```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3 (không phải 0, 1, 2)
```

**Why:** 3 closures cùng reference đến 1 biến `i` (function-scoped)

**Fix:** Dùng `let` (block scope) → mỗi iteration có `i` riêng
```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2 ✅
```

---

#### **2. Memory Leaks với Closures**

**Problem:**
- Closure captures toàn bộ outer scope
- Kể cả variables không dùng trong closure body
- Large objects bị giữ lại → Memory leak

**Scenarios dễ leak:**

**Event listeners không cleanup:**
- Attach listener với closure → DOM element + closure data giữ lại
- Remove element nhưng listener vẫn active
- → Detached DOM + closure data không bị GC

**Large data trong closure scope:**
- Closure chỉ dùng 1 property nhưng capture cả large object
- Entire object bị giữ lại
- Multiply qua nhiều closures → Significant memory

**Prevention:**

**1. Cleanup event listeners:**
- `removeEventListener` khi không cần
- React: Return cleanup function từ `useEffect`
- Use `AbortController` cho multiple listeners

**2. Extract only needed data:**
- Thay vì capture entire object, extract cần thiết trước
- Break reference đến large objects

**3. Nullify references:**
- Set closure reference = null khi done
- Break reachability chain → Allow GC

---

### **Performance Considerations**

**Memory cost:**
- Mỗi closure = Function object + Captured environment
- Nhiều closures = Cost multiply
- Large captured scope = Higher cost

**When acceptable:**
- Event handlers (few instances)
- Module setup (one-time)
- User interactions (not hot paths)

**When to optimize:**
- Tight loops (thousands of closures created)
- Frequent renders (React components)
- Performance-critical paths

**Strategies:**
- Extract to outer scope khi có thể
- Only capture needed variables
- Use `useCallback` (React) cho stable references

---

### **Modern Alternative: # Private Fields (ES2022)**

**Closure for privacy (legacy):**
- Soft privacy (convention-based)
- Higher memory overhead
- Flexible but less performant

**# Private fields (modern):**
- True privacy (syntax-level)
- Engine-optimized
- Better performance

**Trade-off:**
- Closure: More flexible, works everywhere
- # fields: Better performance, only in classes

**Recommendation:** Use # fields trong classes, closures cho functional patterns

---

### **Best Practices**

**1. Hiểu mechanism:**
- Closure = Scope chain + Lexical environment
- Not magic, predictable behavior

**2. Be mindful of captures:**
- Biết closure capture gì
- Minimize captured scope
- Extract needed data only

**3. Always cleanup:**
- Event listeners → removeEventListener
- Timers → clearTimeout/clearInterval
- React → useEffect cleanup

**4. Profile when needed:**
- Chrome DevTools heap snapshots
- Check retained size
- Look for unexpected closures

**5. Choose right tool:**
- Closures: Functional patterns, privacy
- # fields: Class-based privacy
- Modules: Public/private exports
```
```

---


## 5. Prototypal Inheritance (Basic)
### 1.1. Tại Sao Prototypal, Không Phải Classical?

**Classical Inheritance (Java, C++)**:
```
Classes = Blueprints
Objects = Instances
Inheritance = Class hierarchy
```

**Prototypal Inheritance (JavaScript)**:
```
Objects = First-class
Inheritance = Object linking
Dynamic = Runtime changes possible
```

**Tradeoffs**:

| Aspect | Classical | Prototypal |
|--------|-----------|------------|
| Flexibility | Low (rigid hierarchies) | High (dynamic) |
| Mental model | Familiar (class-based) | Less intuitive |
| Performance | Predictable | Can be slower |
| Metaprogramming | Limited | Powerful |

### 1.2. Prototype Chain Mechanics

**Lookup mechanism**:
```
1. Check own properties
2. Check __proto__ (prototype)
3. Repeat до null
4. Return undefined
```

**Performance implications**:
- Shallow chain: Fast (~1-2 lookups)
- Deep chain: Slower (>5 lookups)
- Cache: V8 inline caching helps

**Best practices**:
- Keep chains shallow (< 3 levels)
- Use `hasOwnProperty` khi cần
- Consider Map cho dynamic properties

### 1.3. Object Creation Patterns

#### Pattern 1: Object.create()
```
Purpose: Explicit prototype setting
Pros: Clean, flexible
Cons: More verbose
Use when: Need precise control
```

#### Pattern 2: Constructor Functions
```
Purpose: Factory pattern
Pros: Familiar syntax
Cons: Easy to forget 'new'
Use when: Pre-ES6 codebase
```

#### Pattern 3: ES6 Classes
```
Purpose: Modern syntax
Pros: Familiar to OOP developers
Cons: Syntactic sugar (same prototype underneath)
Use when: Modern projects
```

### 1.4. Property Descriptors

**Attributes**:
- `value`: Property value
- `writable`: Can assign?
- `enumerable`: Show in for...in?
- `configurable`: Can delete/redefine?

**Use cases**:

#### Immutable properties
```
writable: false → read-only
configurable: false → permanent
```

#### Hidden properties
```
enumerable: false → hide from iteration
```

#### Getters/Setters
```
get/set → computed properties
Validation → in setter
Lazy eval → in getter
```

### 1.5. Prototype Pollution Attack

**Vulnerability**: Modify Object.prototype

**Impact**: Affects ALL objects

**Exploit**:
```
Malicious input modifies __proto__
All objects inherit polluted property
Security vulnerability
```

**Protection**:
- Use `Object.create(null)` for maps
- Validate inputs
- Freeze prototypes
- Use Map instead of {}

---

## 6. Type Coercion & Comparison

**== vs ===**

**`===` (Strict Equality): KHÔNG type coercion**
- So sánh type trước, khác type → `false`
- Cùng type → So sánh value

**`==` (Loose Equality): CÓ type coercion**
- Tự động convert type rồi mới so sánh
- **Nguồn gốc của nhiều bugs khó hiểu**

### **Type Coercion của == (Nguy hiểm!)**

```javascript
// Những kết quả unexpected:
5 == '5'          // true (string → number)
0 == false        // true (boolean → number)
'' == 0           // true (string → number)
[] == false       // true (!!)
null == undefined // true (exception)

// Nhưng:
null == 0         // false (exception - không convert)
undefined == 0    // false (exception)

// Absurd:
'' == '0'         // false
0 == ''           // true
0 == '0'          // true
// → '' == '0' là false nhưng cả 2 đều == 0 !?
```

### **Quy tắc Type Coercion (Rối não)**

1. `null == undefined` → `true` (chỉ bằng nhau)
2. Number == String → Convert string to number
3. Boolean == anything → Convert boolean to number (true=1, false=0)
4. Object == Primitive → `obj.valueOf()` hoặc `obj.toString()`

**VÌ VẬY: Luôn dùng ===**

### **Truthy & Falsy**

**8 giá trị Falsy:**
```javascript
false
0, -0
0n (BigInt zero)
'' (empty string)
null
undefined
NaN
```

**Tất cả còn lại là Truthy**, kể cả:
```javascript
'0'        // true (string không rỗng)
'false'    // true
[]         // true (empty array)
{}         // true (empty object)
function(){} // true
```

### **So sánh Objects**

```javascript
const obj1 = { a: 1 };
const obj2 = { a: 1 };

obj1 == obj2   // false (khác reference)
obj1 === obj2  // false

const obj3 = obj1;
obj1 === obj3  // true (cùng reference)
```

**So sánh object by value → Cần deep equality function hoặc thư viện**

### **Object.is() - "Same Value Equality"**

**Khác biệt với `===`:**

```javascript
// 1. NaN
NaN === NaN           // false
Object.is(NaN, NaN)   // true ✅

// 2. +0 vs -0
+0 === -0             // true
Object.is(+0, -0)     // false ✅

// Còn lại giống ===
Object.is(5, 5)       // true
Object.is('a', 'a')   // true
```

### **Best Practice**

```javascript
// ✅ Luôn dùng ===
if (value === 0) { }
if (array.length === 0) { }

// ✅ Exception DUY NHẤT: Check null/undefined cùng lúc
if (value == null) { }  // true cho cả null và undefined
// Tương đương:
if (value === null || value === undefined) { }

// ⚠️ LƯU Ý: Đây là EXCEPTION DUY NHẤT được phép dùng ==
// value == null là idiomatic JavaScript pattern
// Mọi trường hợp khác → Luôn dùng ===

// ✅ Explicit type conversion
if (Number(value) === 0) { }
if (String(value) === '0') { }

// ❌ NEVER dùng == trừ trường hợp null check
if (value == 0) { }  // Unclear intent
```

### **Practical Guideline (Senior Perspective)**

**Khi nào type coercion hữu ích:**

**✅ Explicit coercion (intentional) - GOOD:**
```javascript
Number(userInput)   // String → Number (clear intent)
String(value)       // Anything → String
Boolean(value)      // Truthy/falsy check
+value              // Shorthand String → Number
!!value             // Shorthand Boolean conversion
```

**❌ Implicit coercion (== operator) - BAD:**
```javascript
value == 0          // Unclear what happens
value == ''         // Confusing behavior
```

**Rule of thumb:**
- Explicit coercion: ✅ Clear, intentional → OK to use
- Implicit coercion: ❌ Confusing → Avoid
- Type checking trước: if (typeof x === 'number')

**Senior mindset:**
"Code nên express intent clearly. Implicit coercion hides intent."

---

## 7. Spread/Rest Operators

### **Spread Operator `...` - Expand Iterables**

**Concept**: "Unpack" elements từ iterable (array, string, object)

---

#### **Array Spread**

**1. Combine arrays:**
```javascript
const arr1 = [1, 2];
const arr2 = [3, 4];
const combined = [...arr1, ...arr2]; // [1, 2, 3, 4]
```

**2. Clone array (shallow):**
```javascript
const original = [1, 2, 3];
const clone = [...original];

clone.push(4);
console.log(original); // [1, 2, 3] (unchanged)
```

**3. Convert iterable to array:**
```javascript
const str = 'hello';
const chars = [...str]; // ['h', 'e', 'l', 'l', 'o']

const set = new Set([1, 2, 3]);
const arr = [...set]; // [1, 2, 3]
```

**4. Function arguments:**
```javascript
const numbers = [1, 5, 3, 9, 2];
Math.max(...numbers); // 9
// Equivalent: Math.max(1, 5, 3, 9, 2)
```

---

#### **Object Spread**

**1. Clone object (shallow):**
```javascript
const user = { name: 'Alice', age: 30 };
const clone = { ...user };
```

**2. Merge objects:**
```javascript
const defaults = { theme: 'light', lang: 'en' };
const userPrefs = { theme: 'dark' };
const config = { ...defaults, ...userPrefs };
// { theme: 'dark', lang: 'en' } (userPrefs overrides)
```

**3. Add/override properties:**
```javascript
const user = { name: 'Bob' };
const updated = { ...user, age: 25, name: 'Robert' };
// { name: 'Robert', age: 25 }
```

---

#### **⚠️ Shallow Copy Warning**

**Problem**: Nested objects/arrays share references

```javascript
const original = {
  name: 'Alice',
  address: { city: 'NYC' }
};

const clone = { ...original };

clone.address.city = 'LA';
console.log(original.address.city); // 'LA' ❌ (mutated!)
```

**Why**: Spread chỉ copy top-level, nested objects vẫn là references

**Solutions:**

**1. Nested spread (manual):**
```javascript
const clone = {
  ...original,
  address: { ...original.address }
};
```

**2. structuredClone() (modern):**
```javascript
const clone = structuredClone(original); // Deep clone
```

**3. JSON (naive, lossy):**
```javascript
const clone = JSON.parse(JSON.stringify(original));
// ❌ Loses: functions, undefined, Date, etc.
```

---

### **Rest Parameters `...` - Collect Arguments**

**Concept**: "Pack" multiple elements into array

---

#### **Function Parameters**

**Collect remaining args:**
```javascript
function sum(first, ...rest) {
  console.log(first); // 1
  console.log(rest);  // [2, 3, 4]
  return first + rest.reduce((a, b) => a + b, 0);
}

sum(1, 2, 3, 4); // 10
```

**Replaces `arguments` object:**
```javascript
// ❌ OLD: arguments (array-like, not real array)
function oldWay() {
  const args = Array.from(arguments);
  return args.reduce((a, b) => a + b);
}

// ✅ NEW: rest parameters (real array)
function newWay(...args) {
  return args.reduce((a, b) => a + b);
}
```

**Must be last parameter:**
```javascript
function foo(a, ...rest, b) {} // ❌ SyntaxError
function foo(a, b, ...rest) {} // ✅ OK
```

---

#### **Destructuring with Rest**

**Array destructuring:**
```javascript
const [first, second, ...rest] = [1, 2, 3, 4, 5];
console.log(first);  // 1
console.log(second); // 2
console.log(rest);   // [3, 4, 5]
```

**Object destructuring:**
```javascript
const user = { name: 'Alice', age: 30, city: 'NYC', country: 'USA' };
const { name, ...rest } = user;
console.log(name); // 'Alice'
console.log(rest); // { age: 30, city: 'NYC', country: 'USA' }
```

---

### **Practical Patterns**

#### **Pattern 1: Immutable Updates**
```javascript
// Add item to array
const todos = [{ id: 1, text: 'Learn JS' }];
const newTodos = [...todos, { id: 2, text: 'Learn TS' }];

// Remove item
const filtered = todos.filter(todo => todo.id !== 1);

// Update nested object
const state = {
  user: { name: 'Alice', settings: { theme: 'light' } }
};

const updated = {
  ...state,
  user: {
    ...state.user,
    settings: {
      ...state.user.settings,
      theme: 'dark'
    }
  }
};
```

#### **Pattern 2: Flexible Function APIs**
```javascript
function createUser(name, ...options) {
  const defaults = { active: true, role: 'user' };
  const config = Object.assign(defaults, ...options);
  return { name, ...config };
}

createUser('Alice', { role: 'admin' }, { premium: true });
// { name: 'Alice', active: true, role: 'admin', premium: true }
```

#### **Pattern 3: Props Forwarding (React)**
```javascript
function Button({ onClick, children, ...rest }) {
  return (
    
      {children}
    
  );
}

// Usage: All extra props forwarded to 

  Click Me

```

---

### **Performance Considerations**

**Spread is O(n):**
```javascript
// ❌ BAD: O(n²) trong loop
const result = [];
for (let i = 0; i < 1000; i++) {
  result = [...result, i]; // Copy entire array mỗi iteration
}

// ✅ GOOD: O(n) - mutate then freeze
const result = [];
for (let i = 0; i < 1000; i++) {
  result.push(i);
}
Object.freeze(result); // Immutable sau khi build
```

**Large objects:**
- Spread object với 1000+ keys = noticeable cost
- Consider mutable update + freeze
- Or use Immer library

---

### **Common Mistakes**

**1. Spreading non-iterables:**
```javascript
const obj = { a: 1 };
const arr = [...obj]; // ❌ TypeError: obj is not iterable

// Only objects spread into objects, iterables into arrays
```

**2. Thinking spread is deep:**
```javascript
const original = { nested: { value: 1 } };
const clone = { ...original };
clone.nested.value = 2;
console.log(original.nested.value); // 2 ❌ (shallow!)
```

**3. Order matters in merging:**
```javascript
const merged = { ...obj1, ...obj2 };
// obj2 properties override obj1 (right-to-left priority)
```

---

## 8. Array Methods (Iteration)

### **Core Iteration Methods**

---

#### **map() - Transform Elements**

**Purpose**: Tạo array mới bằng cách transform mỗi element

```javascript
const numbers = [1, 2, 3];
const doubled = numbers.map(n => n * 2); // [2, 4, 6]

// With index
const indexed = numbers.map((n, i) => `${i}: ${n}`);
// ['0: 1', '1: 2', '2: 3']
```

**Common use case**: Extract properties
```javascript
const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
];
const names = users.map(u => u.name); // ['Alice', 'Bob']
```

**⚠️ Return value matters:**
```javascript
// ❌ BAD: Implicit return undefined
const result = [1, 2, 3].map(n => {
  n * 2; // No return!
});
console.log(result); // [undefined, undefined, undefined]

// ✅ GOOD
const result = [1, 2, 3].map(n => n * 2);
```

---

#### **filter() - Select Elements**

**Purpose**: Tạo array mới chỉ chứa elements thỏa điều kiện

```javascript
const numbers = [1, 2, 3, 4, 5];
const evens = numbers.filter(n => n % 2 === 0); // [2, 4]

// Complex predicate
const users = [
  { name: 'Alice', age: 25, active: true },
  { name: 'Bob', age: 17, active: false },
  { name: 'Charlie', age: 30, active: true }
];

const activeAdults = users.filter(u => u.active && u.age >= 18);
// [{ name: 'Alice', ... }, { name: 'Charlie', ... }]
```

**Truthiness-based filtering:**
```javascript
const values = [0, 1, '', 'hello', null, 'world', undefined];
const truthy = values.filter(Boolean); // [1, 'hello', 'world']
```

---

#### **reduce() - Accumulate to Single Value**

**Purpose**: "Reduce" array thành 1 value (number, object, array, etc.)

**Syntax**: `arr.reduce((accumulator, current, index, array) => newAcc, initialValue)`

**Examples:**

**1. Sum:**
```javascript
const numbers = [1, 2, 3, 4];
const sum = numbers.reduce((acc, n) => acc + n, 0); // 10
```

**2. Object from array:**
```javascript
const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
];

const byId = users.reduce((acc, user) => {
  acc[user.id] = user;
  return acc;
}, {});
// { 1: { id: 1, name: 'Alice' }, 2: { id: 2, name: 'Bob' } }
```

**3. Flatten array:**
```javascript
const nested = [[1, 2], [3, 4], [5]];
const flat = nested.reduce((acc, arr) => acc.concat(arr), []);
// [1, 2, 3, 4, 5]

// Modern: Use flat()
const flat = nested.flat(); // Easier!
```

**4. Count occurrences:**
```javascript
const fruits = ['apple', 'banana', 'apple', 'orange', 'banana', 'apple'];
const count = fruits.reduce((acc, fruit) => {
  acc[fruit] = (acc[fruit] || 0) + 1;
  return acc;
}, {});
// { apple: 3, banana: 2, orange: 1 }
```

**⚠️ Initial value matters:**
```javascript
// ❌ No initial value với empty array
[].reduce((acc, n) => acc + n); // TypeError

// ✅ Always provide initial value
[].reduce((acc, n) => acc + n, 0); // 0 (safe)
```

---

#### **find() / findIndex() - Locate Element**

**find()**: Return first matching element (or undefined)
```javascript
const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
];

const user = users.find(u => u.id === 2);
// { id: 2, name: 'Bob' }

const notFound = users.find(u => u.id === 999);
// undefined
```

**findIndex()**: Return index of first match (or -1)
```javascript
const index = users.findIndex(u => u.id === 2); // 1
const notFound = users.findIndex(u => u.id === 999); // -1
```

---

#### **some() / every() - Boolean Tests**

**some()**: At least 1 element matches? (OR logic)
```javascript
const numbers = [1, 3, 5, 8, 9];
const hasEven = numbers.some(n => n % 2 === 0); // true (8 is even)
```

**every()**: All elements match? (AND logic)
```javascript
const allOdd = numbers.every(n => n % 2 !== 0); // false (8 is even)
```

**Use case: Validation**
```javascript
const users = [
  { name: 'Alice', age: 25 },
  { name: 'Bob', age: 30 }
];

const allAdults = users.every(u => u.age >= 18); // true
const hasMinor = users.some(u => u.age < 18);   // false
```

---

### **Functional Patterns**

#### **Chaining Methods**

**Readable pipeline:**
```javascript
const users = [
  { name: 'Alice', age: 25, active: true },
  { name: 'Bob', age: 17, active: false },
  { name: 'Charlie', age: 30, active: true },
  { name: 'Dave', age: 22, active: true }
];

const result = users
  .filter(u => u.active)           // [Alice, Charlie, Dave]
  .filter(u => u.age >= 21)        // [Alice, Charlie, Dave]
  .map(u => u.name)                // ['Alice', 'Charlie', 'Dave']
  .sort();                         // ['Alice', 'Charlie', 'Dave']

console.log(result); // ['Alice', 'Charlie', 'Dave']
```

**⚠️ Performance consideration:**
- Multiple filters = multiple passes
- For large arrays, consider single reduce

**Optimized (single pass):**
```javascript
const result = users.reduce((acc, u) => {
  if (u.active && u.age >= 21) {
    acc.push(u.name);
  }
  return acc;
}, []).sort();
```

---

#### **Immutability Pattern**

**Never mutate, always return new:**
```javascript
// ❌ BAD: Mutates original
const addItem = (arr, item) => {
  arr.push(item); // Mutation!
  return arr;
};

// ✅ GOOD: Returns new array
const addItem = (arr, item) => [...arr, item];

// ✅ GOOD: Functional update
const updateItem = (arr, id, updates) =>
  arr.map(item => item.id === id ? { ...item, ...updates } : item);

// ✅ GOOD: Remove item
const removeItem = (arr, id) => arr.filter(item => item.id !== id);
```

---

### **Performance Considerations**

#### **Time Complexity**

| Method | Complexity | Notes |
|--------|-----------|-------|
| map | O(n) | Single pass |
| filter | O(n) | Single pass |
| reduce | O(n) | Single pass |
| find | O(n) worst | Stops early if found |
| some/every | O(n) worst | Short-circuits |
| sort | O(n log n) | In-place mutation |

#### **Memory Usage**

**map/filter create new arrays:**
```javascript
// ❌ Memory-intensive cho large arrays
const huge = Array.from({ length: 1_000_000 }, (_, i) => i);
const result = huge.map(n => n * 2).filter(n => n > 100);
// 2 intermediate arrays created!
```

**Better: Generator functions cho lazy evaluation:**
```javascript
function* lazyMap(arr, fn) {
  for (const item of arr) yield fn(item);
}

function* lazyFilter(iterable, predicate) {
  for (const item of iterable) {
    if (predicate(item)) yield item;
  }
}

// Single pass, no intermediate arrays
const result = Array.from(
  lazyFilter(
    lazyMap(huge, n => n * 2),
    n => n > 100
  )
);
```

---

#### **When to Optimize**

**Don't optimize prematurely:**
- Arrays < 10K elements: Chaining is fine
- Readable code > micro-optimizations

**Optimize when:**
- Arrays > 100K elements
- Hot paths (called frequently)
- Real performance issues measured

**Techniques:**
- Use for loops (faster but less readable)
- Single reduce instead of chain
- Lazy evaluation với generators
- Consider native Set/Map operations

---

### **Common Mistakes**

**1. Forgetting return trong map:**
```javascript
// ❌
[1, 2, 3].map(n => { n * 2 }); // [undefined, undefined, undefined]

// ✅
[1, 2, 3].map(n => n * 2);
[1, 2, 3].map(n => { return n * 2 });
```

**2. Mutating trong reduce:**
```javascript
// ❌ Mutates accumulator
const result = arr.reduce((acc, item) => {
  acc.push(item); // Mutation!
  return acc;
}, []);

// ✅ Return new array
const result = arr.reduce((acc, item) => [...acc, item], []);
// But for this case, just use: const result = [...arr];
```

**3. Missing initial value trong reduce:**
```javascript
// ❌ Breaks với empty array
[].reduce((acc, n) => acc + n); // TypeError

// ✅
[].reduce((acc, n) => acc + n, 0); // 0
```

---

## 9. Object Methods (Utility)

### **Object Static Methods**

---

#### **Object.keys() / values() / entries()**

**Object.keys()**: Array of own enumerable property names
```javascript
const user = { name: 'Alice', age: 30, city: 'NYC' };
Object.keys(user); // ['name', 'age', 'city']
```

**Object.values()**: Array of property values
```javascript
Object.values(user); // ['Alice', 30, 'NYC']
```

**Object.entries()**: Array of [key, value] pairs
```javascript
Object.entries(user);
// [['name', 'Alice'], ['age', 30], ['city', 'NYC']]

// Use case: Convert object to Map
const map = new Map(Object.entries(user));
```

**Iteration pattern:**
```javascript
// Instead of for...in (includes prototype chain)
for (const key in user) {
  if (user.hasOwnProperty(key)) {
    console.log(key, user[key]);
  }
}

// ✅ Better: Object.entries
for (const [key, value] of Object.entries(user)) {
  console.log(key, value);
}
```

---

#### **Object.assign() - Shallow Merge**

**Purpose**: Copy properties từ source objects vào target

**Syntax**: `Object.assign(target, ...sources)`

```javascript
const target = { a: 1, b: 2 };
const source = { b: 3, c: 4 };

Object.assign(target, source);
console.log(target); // { a: 1, b: 3, c: 4 } (mutated!)
```

**⚠️ Mutates target!**

**Immutable pattern: Empty object as target**
```javascript
const obj1 = { a: 1, b: 2 };
const obj2 = { b: 3, c: 4 };

const merged = Object.assign({}, obj1, obj2);
// { a: 1, b: 3, c: 4 }
// obj1 and obj2 unchanged
```

**Modern alternative: Spread (preferred)**
```javascript
const merged = { ...obj1, ...obj2 };
// Same result, cleaner syntax
```

**Shallow copy nature:**
```javascript
const original = { nested: { value: 1 } };
const clone = Object.assign({}, original);

clone.nested.value = 2;
console.log(original.nested.value); // 2 ❌ (shallow!)
```

---

#### **Object.freeze() - Immutability**

**Purpose**: Make object immutable (cannot add/delete/modify properties)

```javascript
const user = { name: 'Alice', age: 30 };
Object.freeze(user);

user.age = 31;        // ❌ Silently fails (strict: TypeError)
user.city = 'NYC';    // ❌ Cannot add
delete user.name;     // ❌ Cannot delete

console.log(user); // { name: 'Alice', age: 30 } (unchanged)
```

**Check if frozen:**
```javascript
Object.isFrozen(user); // true
```

**⚠️ Shallow freeze:**
```javascript
const obj = {
  name: 'Alice',
  address: { city: 'NYC' }
};

Object.freeze(obj);

obj.name = 'Bob';           // ❌ Cannot modify
obj.address.city = 'LA';    // ✅ CAN modify! (nested not frozen)
console.log(obj.address.city); // 'LA'
```

**Deep freeze (manual):**
```javascript
function deepFreeze(obj) {
  Object.freeze(obj);
  
  Object.values(obj).forEach(value => {
    if (typeof value === 'object' && value !== null) {
      deepFreeze(value);
    }
  });
  
  return obj;
}

const obj = { nested: { value: 1 } };
deepFreeze(obj);
obj.nested.value = 2; // ❌ TypeError in strict mode
```

---

#### **Object.seal() - Partial Immutability**

**Purpose**: Cannot add/delete properties, nhưng có thể modify existing

```javascript
const user = { name: 'Alice', age: 30 };
Object.seal(user);

user.age = 31;        // ✅ Can modify
user.city = 'NYC';    // ❌ Cannot add
delete user.name;     // ❌ Cannot delete

console.log(user); // { name: 'Alice', age: 31 }
```

**Check if sealed:**
```javascript
Object.isSealed(user); // true
```

**Comparison:**

| Method | Add | Delete | Modify |
|--------|-----|--------|--------|
| `freeze()` | ❌ | ❌ | ❌ |
| `seal()` | ❌ | ❌ | ✅ |
| Normal object | ✅ | ✅ | ✅ |

**Use cases:**
- `freeze()`: Constants, config objects
- `seal()`: Fixed shape objects (TypeScript-like)

---

#### **Object.create() - Prototype Setting**

**Purpose**: Tạo object mới với specified prototype

```javascript
const personProto = {
  greet() {
    console.log(`Hello, ${this.name}`);
  }
};

const alice = Object.create(personProto);
alice.name = 'Alice';
alice.greet(); // "Hello, Alice"

console.log(alice.__proto__ === personProto); // true
```

**Create object without prototype:**
```javascript
const pureObject = Object.create(null);
// No prototype chain, không có Object.prototype methods

console.log(pureObject.toString); // undefined
console.log(pureObject.__proto__); // undefined

// Use case: Clean dictionary/map (no property collision)
```

**With property descriptors:**
```javascript
const obj = Object.create(proto, {
  name: {
    value: 'Alice',
    writable: true,
    enumerable: true,
    configurable: true
  }
});
```

---

#### **hasOwnProperty() - Property Check**

**Purpose**: Check if property belongs to object itself (not inherited)

```javascript
const parent = { inherited: 'value' };
const child = Object.create(parent);
child.own = 'mine';

console.log(child.inherited); // 'value' (accessible)
console.log(child.own); // 'mine'

child.hasOwnProperty('own'); // true
child.hasOwnProperty('inherited'); // false (inherited)
child.hasOwnProperty('toString'); // false (from Object.prototype)
```

**⚠️ hasOwnProperty can be shadowed:**
```javascript
const obj = {
  hasOwnProperty: 'not a function' // Shadowed!
};

obj.hasOwnProperty('key'); // ❌ TypeError

// ✅ Safe way
Object.prototype.hasOwnProperty.call(obj, 'key');

// ✅ Modern alternative (ES2022)
Object.hasOwn(obj, 'key');
```

**Use case: for...in loops**
```javascript
const obj = { a: 1, b: 2 };

// ❌ BAD: Iterates prototype chain
for (const key in obj) {
  console.log(key, obj[key]);
}

// ✅ GOOD: Filter own properties
for (const key in obj) {
  if (Object.hasOwn(obj, key)) {
    console.log(key, obj[key]);
  }
}

// ✅ BETTER: Use Object.keys/entries
Object.keys(obj).forEach(key => {
  console.log(key, obj[key]);
});
```

---

#### **in Operator vs hasOwnProperty**

**`in` operator**: Checks entire prototype chain
```javascript
const parent = { inherited: true };
const child = Object.create(parent);
child.own = true;

'own' in child; // true
'inherited' in child; // true (checks prototype)
'toString' in child; // true (Object.prototype)
```

**hasOwnProperty**: Only own properties
```javascript
child.hasOwnProperty('own'); // true
child.hasOwnProperty('inherited'); // false
```

**When to use:**
- `in`: Check if property accessible (including inherited)
- `hasOwnProperty`: Check if property is own (not inherited)

---

### **Property Descriptors**

#### **Object.getOwnPropertyDescriptor()**

**Get descriptor of property:**
```javascript
const obj = { name: 'Alice' };
const descriptor = Object.getOwnPropertyDescriptor(obj, 'name');

console.log(descriptor);
// {
//   value: 'Alice',
//   writable: true,
//   enumerable: true,
//   configurable: true
// }
```

#### **Object.defineProperty()**

**Define property with custom descriptor:**
```javascript
const obj = {};

Object.defineProperty(obj, 'readOnly', {
  value: 42,
  writable: false,
  enumerable: true,
  configurable: false
});

obj.readOnly = 100; // ❌ Silently fails (strict: TypeError)
console.log(obj.readOnly); // 42 (unchanged)

delete obj.readOnly; // ❌ Cannot delete (configurable: false)
```

**Getters/Setters:**
```javascript
const user = {
  firstName: 'Alice',
  lastName: 'Smith'
};

Object.defineProperty(user, 'fullName', {
  get() {
    return `${this.firstName} ${this.lastName}`;
  },
  set(value) {
    [this.firstName, this.lastName] = value.split(' ');
  },
  enumerable: true,
  configurable: true
});

console.log(user.fullName); // "Alice Smith"
user.fullName = 'Bob Jones';
console.log(user.firstName); // "Bob"
```

---

### **Practical Patterns**

#### **Pattern 1: Config Objects**
```javascript
const CONFIG = Object.freeze({
  API_URL: 'https://api.example.com',
  TIMEOUT: 5000,
  RETRY_ATTEMPTS: 3
});

// Cannot accidentally modify
CONFIG.API_URL = 'https://evil.com'; // ❌ Silently fails
```

#### **Pattern 2: Enums (Pre-TypeScript)**
```javascript
const STATUS = Object.freeze({
  PENDING: 'pending',
  APPROVED: 'approved',
  REJECTED: 'rejected'
});

function processOrder(status) {
  if (status === STATUS.APPROVED) {
    // ...
  }
}
```

#### **Pattern 3: Object as Dictionary**
```javascript
// ❌ BAD: Object with prototype
const cache = {};
cache['__proto__'] = 'polluted'; // Pollution risk

// ✅ GOOD: No prototype
const cache = Object.create(null);
cache['__proto__'] = 'safe'; // Just a normal property

// ✅ BETTER: Use Map
const cache = new Map();
cache.set('__proto__', 'safe');
```

---

## 10. Error Handling (Imperative)

### **try/catch/finally**

#### **Basic Syntax**

```javascript
try {
  // Code that might throw
  const result = riskyOperation();
  console.log(result);
} catch (error) {
  // Handle error
  console.error('Operation failed:', error.message);
} finally {
  // Always executes (cleanup)
  cleanup();
}
```

**Execution flow:**
1. Execute `try` block
2. If error → Jump to `catch`, skip remaining try code
3. If no error → Skip `catch`
4. **Always** execute `finally` (even if return in try/catch)

---

#### **finally Always Runs**

**Even with return:**
```javascript
function test() {
  try {
    return 'try';
  } finally {
    console.log('Finally runs'); // Executes!
  }
}

test(); // Logs "Finally runs", returns 'try'
```

**finally return overrides try return:**
```javascript
function test() {
  try {
    return 'try';
  } finally {
    return 'finally'; // Overrides!
  }
}

test(); // Returns 'finally' (not 'try')
```

**Use case: Cleanup**
```javascript
function readFile(path) {
  const file = openFile(path);
  
  try {
    return processFile(file);
  } catch (error) {
    console.error('Failed to process:', error);
    throw error;
  } finally {
    closeFile(file); // Always cleanup, even if error
  }
}
```

---

### **Error Types**

#### **Built-in Error Types**

**1. Error (Base class)**
```javascript
throw new Error('Something went wrong');
```

**2. SyntaxError**
```javascript
// eval('invalid javascript code'); // SyntaxError
JSON.parse('invalid json'); // SyntaxError
```

**3. ReferenceError**
```javascript
console.log(undeclaredVariable); // ReferenceError
```

**4. TypeError**
```javascript
null.toString(); // TypeError: Cannot read property
(123).toUpperCase(); // TypeError: Not a function
```

**5. RangeError**
```javascript
new Array(-1); // RangeError: Invalid array length
(123).toFixed(101); // RangeError: toFixed digits out of range
```

**6. URIError**
```javascript
decodeURIComponent('%'); // URIError: Malformed URI
```

---

### **Custom Errors**

#### **Extend Error Class**

```javascript
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
    
    // Maintain stack trace (V8 only)
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, ValidationError);
    }
  }
}

// Usage
function validateUser(user) {
  if (!user.email) {
    throw new ValidationError('Email is required', 'email');
  }
  if (user.age < 0) {
    throw new ValidationError('Age must be positive', 'age');
  }
}

try {
  validateUser({ email: '', age: -5 });
} catch (error) {
  if (error instanceof ValidationError) {
    console.error(`Validation failed on ${error.field}: ${error.message}`);
  } else {
    throw error; // Re-throw unknown errors
  }
}
```

---

#### **Error Hierarchy**

```javascript
class AppError extends Error {
  constructor(message) {
    super(message);
    this.name = 'AppError';
  }
}

class DatabaseError extends AppError {
  constructor(message, query) {
    super(message);
    this.name = 'DatabaseError';
    this.query = query;
  }
}

class NetworkError extends AppError {
  constructor(message, statusCode) {
    super(message);
    this.name = 'NetworkError';
    this.statusCode = statusCode;
  }
}

// Centralized error handling
function handleError(error) {
  if (error instanceof DatabaseError) {
    console.error('DB Error:', error.message, error.query);
    // Log to database error tracking
  } else if (error instanceof NetworkError) {
    console.error('Network Error:', error.statusCode, error.message);
    // Retry logic
  } else if (error instanceof AppError) {
    console.error('App Error:', error.message);
  } else {
    console.error('Unknown Error:', error);
    // Alert on-call engineer
  }
}
```

---

### **Error Handling Patterns**

#### **Pattern 1: Fail Fast**

**Validate early, throw immediately:**
```javascript
function processOrder(order) {
  // ✅ Validate at entry point
  if (!order) {
    throw new TypeError('Order is required');
  }
  if (!order.items || order.items.length === 0) {
    throw new ValidationError('Order must have items', 'items');
  }
  if (order.total < 0) {
    throw new ValidationError('Total cannot be negative', 'total');
  }
  
  // Proceed with valid data
  return calculateShipping(order);
}
```

**Benefits:**
- Errors caught early (closer to cause)
- Invalid state never propagates
- Easier debugging (clear stack trace)

---

#### **Pattern 2: Guard Clauses**

**Multiple returns for error cases:**
```javascript
function divide(a, b) {
  if (typeof a !== 'number') {
    throw new TypeError('First argument must be number');
  }
  if (typeof b !== 'number') {
    throw new TypeError('Second argument must be number');
  }
  if (b === 0) {
    throw new RangeError('Cannot divide by zero');
  }
  
  // Happy path at end (no nesting)
  return a / b;
}
```

**vs Deep nesting:**
```javascript
// ❌ BAD: Nested if hell
function divide(a, b) {
  if (typeof a === 'number') {
    if (typeof b === 'number') {
      if (b !== 0) {
        return a / b;
      } else {
        throw new Error('Cannot divide by zero');
      }
    } else {
      throw new TypeError('Second argument must be number');
    }
  } else {
    throw new TypeError('First argument must be number');
  }
}
```

---

#### **Pattern 3: Error Context**

**Attach useful debugging info:**
```javascript
class APIError extends Error {
  constructor(message, context) {
    super(message);
    this.name = 'APIError';
    this.context = context; // url, method, status, response, etc.
    this.timestamp = new Date().toISOString();
  }
}

async function fetchUser(id) {
  const url = `/api/users/${id}`;
  
  try {
    const response = await fetch(url);
    
    if (!response.ok) {
      throw new APIError('Failed to fetch user', {
        url,
        method: 'GET',
        status: response.status,
        statusText: response.statusText
      });
    }
    
    return await response.json();
  } catch (error) {
    // Log full context
    console.error('API Error:', error.message, error.context);
    throw error;
  }
}
```

---

#### **Pattern 4: Wrap and Rethrow**

**Add context while preserving original error:**
```javascript
function processUserData(userId) {
  try {
    const user = fetchUser(userId);
    return transformData(user);
  } catch (error) {
    // Wrap with context
    const wrappedError = new Error(`Failed to process user ${userId}`);
    wrappedError.cause = error; // Preserve original (ES2022)
    throw wrappedError;
  }
}

// Caller can access both
try {
  processUserData(123);
} catch (error) {
  console.error(error.message); // "Failed to process user 123"
  console.error(error.cause); // Original error
}
```

---

### **Defensive Programming**

#### **Type Checking**

```javascript
function calculateTotal(items) {
  // Defensive: Check input type
  if (!Array.isArray(items)) {
    throw new TypeError('Items must be an array');
  }
  
  return items.reduce((total, item) => {
    // Defensive: Check item structure
    if (typeof item.price !== 'number') {
      throw new TypeError(`Invalid price for item ${item.name}`);
    }
    return total + item.price;
  }, 0);
}
```

#### **Null/Undefined Checks**

```javascript
function getUserEmail(user) {
  // ✅ Explicit null check
  if (!user) {
    throw new TypeError('User is required');
  }
  if (!user.email) {
    throw new ValidationError('User email is missing', 'email');
  }
  
  return user.email.toLowerCase();
}
```

#### **Default Values**

```javascript
function greet(name = 'Guest') {
  // Defensive: Default parameter
  return `Hello, ${name}`;
}

function processConfig(config = {}) {
  // Defensive: Default to empty object
  const timeout = config.timeout ?? 5000; // Nullish coalescing
  const retries = config.retries ?? 3;
  
  return { timeout, retries };
}
```

---

### **Error Handling Best Practices**

#### **1. Don't Catch Everything**

```javascript
// ❌ BAD: Swallow all errors
try {
  riskyOperation();
} catch (error) {
  console.log('Error occurred'); // No info, no rethrow!
}

// ✅ GOOD: Handle specific errors, rethrow unknown
try {
  riskyOperation();
} catch (error) {
  if (error instanceof ValidationError) {
    // Handle expected error
    showUserError(error.message);
  } else {
    // Unexpected error - rethrow
    console.error('Unexpected error:', error);
    throw error;
  }
}
```

#### **2. Log Before Throwing**

```javascript
// ✅ Log full context before throwing
function criticalOperation() {
  try {
    return dangerousCode();
  } catch (error) {
    // Log with context
    console.error('Critical operation failed', {
      error: error.message,
      stack: error.stack,
      timestamp: Date.now(),
      user: getCurrentUser()
    });
    
    // Then throw (or handle)
    throw error;
  }
}
```

#### **3. Never Ignore Errors**

```javascript
// ❌ BAD: Empty catch
try {
  riskyOperation();
} catch (error) {
  // Silent failure - terrible!
}

// ✅ GOOD: At minimum, log
try {
  riskyOperation();
} catch (error) {
  console.error('Operation failed:', error);
  // Optionally: Send to error tracking service
}
```

#### **4. Use finally for Cleanup**

```javascript
// ✅ Always cleanup resources
function processFile(path) {
  const file = openFile(path);
  
  try {
    return processContents(file);
  } catch (error) {
    console.error('Failed to process file:', error);
    throw error;
  } finally {
    // Always runs - even if error or return
    closeFile(file);
  }
}
```

---

### **Production Error Handling**

#### **Global Error Handlers**

**Browser:**
```javascript
window.addEventListener('error', (event) => {
  console.error('Uncaught error:', event.error);
  // Send to error tracking (Sentry, etc.)
  errorTracker.captureException(event.error);
});

window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled promise rejection:', event.reason);
  errorTracker.captureException(event.reason);
});
```

**Node.js:**
```javascript
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  // Log and gracefully shutdown
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
  // Log but don't crash (handle gracefully)
});
```

---

### **Quick Reference**

| Error Type | When Thrown |
|------------|-------------|
| `Error` | Generic error |
| `SyntaxError` | Invalid syntax (parse error) |
| `ReferenceError` | Undefined variable |
| `TypeError` | Wrong type operation |
| `RangeError` | Value out of range |
| `URIError` | Invalid URI handling |

**Best Practices:**
- ✅ Fail fast with clear errors
- ✅ Use custom error types for domain errors
- ✅ Always cleanup in finally
- ✅ Log before throwing
- ✅ Never swallow errors silently
- ✅ Handle specific errors, rethrow unknown
- ✅ Add context to errors for debugging