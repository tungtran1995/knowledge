# Object-Oriented Programming - OOP trong JavaScript

> **Mục tiêu**: Hiểu sâu về OOP trong JavaScript, từ prototypal inheritance đến design patterns và SOLID principles trong production.

---

## 📚 Table of Contents

1. [Prototypal Inheritance - JavaScript's DNA](#1-prototypal-inheritance---javascripts-dna)
2. [Class Syntax - Modern OOP](#2-class-syntax---modern-oop)
3. [Design Patterns in JavaScript](#3-design-patterns-in-javascript)
4. [SOLID Principles](#4-solid-principles)
5. [Composition vs Inheritance](#5-composition-vs-inheritance)
6. [Production Best Practices](#6-production-best-practices)

---

## 1. Prototypal Inheritance - JavaScript's DNA

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

## 2. Class Syntax - Modern OOP

### 2.1. Class Advantages

**Readability**:
```
Clear intent → "This is a class"
Familiar syntax → Java/C++ developers
Grouped methods → better organization
```

**Features**:
- Constructor
- Instance methods
- Static methods
- Getters/Setters
- Private fields (#)
- Inheritance (extends)

**Limitations** (Syntactic sugar):
```
Still prototypal underneath
Not true classes
No multiple inheritance
```

### 2.2. Private Fields Evolution

#### Era 1: Convention (Underscore)
```
_private → "Don't touch"
No enforcement
Just social contract
```

#### Era 2: Closures
```
Variables in constructor
Hidden from outside
Memory overhead
```

#### Era 3: WeakMap
```
External storage
Auto-cleanup
Complex syntax
```

#### Era 4: # Private Fields (ES2022)
```
True privacy
Syntax level
Performance optimized
```

**Best practice**: Use # for new code

### 2.3. Static Members

**Purpose**:
- Utility methods
- Factory methods
- Configuration
- Shared state

**When to use**:
- Doesn't need instance state
- Belongs to class concept
- Shared across instances

**Anti-pattern**: Using static for global state

### 2.4. Inheritance with extends

**Super keyword**:
```
super() → Call parent constructor
super.method() → Call parent method
```

**Best practices**:
- Call super() first in constructor
- Don't overuse inheritance (prefer composition)
- Keep hierarchies shallow (< 3 levels)

**Pitfalls**:
- Forgetting super() → Error
- Multiple inheritance → Not supported (use mixins)

---

## 3. Design Patterns in JavaScript

### 3.1. Creational Patterns

#### Singleton Pattern

**Purpose**: One instance globally

**Use cases**:
- Database connections
- Configuration
- Loggers
- Cache

**Implementation strategies**:
```
1. Module pattern (ESM exports singleton)
2. Class với static instance
3. Closure
```

**Considerations**:
- Testing challenges (global state)
- Hidden dependencies
- Alternative: Dependency injection

#### Factory Pattern

**Purpose**: Encapsulate object creation

**When to use**:
- Complex initialization
- Type decision at runtime
- Hide concrete classes

**Variants**:
- Simple Factory: One factory
- Factory Method: Subclass decides
- Abstract Factory: Factories of factories

**Example applications**:
- React.createElement
- Document.createElement
- Database adapters

#### Builder Pattern

**Purpose**: Construct complex objects step-by-step

**When to use**:
- Many optional parameters
- Validation during construction
- Immutable objects

**Benefits**:
- Fluent API
- Self-documenting
- Type-safe configuration

### 3.2. Structural Patterns

#### Adapter Pattern

**Purpose**: Make incompatible interfaces compatible

**Use cases**:
- Third-party libraries
- Legacy code integration
- API versioning

**Real-world examples**:
- Axios → Fetch adapter
- Different database drivers
- Platform-specific APIs

#### Decorator Pattern

**Purpose**: Add behavior dynamically

**JavaScript implementations**:
```
1. Higher-order functions
2. Class decorators (@)
3. Object composition
```

**Use cases**:
- Logging
- Caching
- Validation
- Authorization

**Benefits over inheritance**:
- Runtime composition
- Multiple decorators
- Avoid class explosion

#### Proxy Pattern

**Purpose**: Control access to object

**JavaScript native**: Proxy API

**Use cases**:
- Validation
- Lazy loading
- Access control
- Reactive systems (Vue 3)

**Performance**: Slight overhead, but negligible for most cases

### 3.3. Behavioral Patterns

#### Observer Pattern

**Purpose**: One-to-many dependency

**JavaScript implementations**:
- EventEmitter (Node.js)
- DOM Events
- RxJS Observables
- Custom implementations

**Use cases**:
- Event systems
- Data binding
- MVC/MVVM
- Pub-Sub messaging

**Considerations**:
- Memory leaks (forgotten listeners)
- Order of notification
- Synchronous vs asynchronous

#### Strategy Pattern

**Purpose**: Interchangeable algorithms

**When to use**:
- Multiple algorithms for same task
- Avoid conditionals
- Runtime selection

**Examples**:
- Sorting algorithms
- Payment methods
- Compression algorithms
- Validation strategies

**Benefits**:
- Open/closed principle
- Easy testing
- Clear separation

#### Command Pattern

**Purpose**: Encapsulate requests as objects

**Use cases**:
- Undo/Redo
- Task queues
- Macro recording
- Transactional operations

**Benefits**:
- Decouple sender/receiver
- Queueable
- Loggable

#### Module Pattern

**Purpose**: Encapsulation + namespacing

**JavaScript evolution**:
```
1. IIFE modules (old)
2. CommonJS (Node.js)
3. AMD (RequireJS)
4. ESM (modern)
```

**Benefits**:
- Private variables
- Public API
- Avoid global pollution

---

## 4. SOLID Principles

### 4.1. Single Responsibility Principle (SRP)

**Definition**: Class should have one reason to change

**Bad signs**:
- Class name với "and"
- Too many methods
- Mix concerns (e.g., User + UserRepository)

**Refactoring**:
```
Split responsibilities
Extract classes
Delegate to collaborators
```

**Benefits**:
- Easier testing
- Better reuse
- Clear dependencies

### 4.2. Open/Closed Principle (OCP)

**Definition**: Open for extension, closed for modification

**Techniques**:

#### 1. Inheritance
```
Extend base class
Override methods
Add new behavior
```

#### 2. Composition
```
Inject dependencies
Strategy pattern
Decorator pattern
```

#### 3. Plugins/Middleware
```
Plugin architecture
Extensible framework
Hook system
```

**Real-world examples**:
- Express middleware
- Babel plugins
- Webpack loaders

### 4.3. Liskov Substitution Principle (LSP)

**Definition**: Subtype должен заменять parent type

**Violations**:
- Strengthen preconditions
- Weaken postconditions
- Throw unexpected errors
- Change behavior unexpectedly

**Classic example**: Square/Rectangle problem

**Solution**:
- Composition over inheritance
- Interface segregation
- Behavioral contracts

### 4.4. Interface Segregation Principle (ISP)

**Definition**: Many specific interfaces > one general

**JavaScript application** (no interfaces):
```
Duck typing
Small, focused mixins
Minimal protocols
```

**Benefits**:
- Smaller dependencies
- Easier testing
- Clear contracts

**Anti-pattern**: Fat interface/class

### 4.5. Dependency Inversion Principle (DIP)

**Definition**: Depend on abstractions, not concrete

**Techniques**:

#### 1. Dependency Injection
```
Constructor injection
Parameter injection
Property injection
```

#### 2. Factory Functions
```
Return implementations
Hide concretions
Swap easily
```

#### 3. Inversion of Control
```
Framework calls you
Hollywood principle
Plugin architecture
```

**Benefits**:
- Testability (inject mocks)
- Flexibility
- Decoupling

---

## 5. Composition vs Inheritance

### 5.1. The Problem with Inheritance

**Issues**:

#### 1. Fragile Base Class
```
Change parent → break children
Ripple effects
Hard to predict impact
```

#### 2. Gorilla-Banana Problem
```
"Want banana, get gorilla +  jungle"
Unwanted dependencies
Cannot pick parts
```

#### 3. Diamond Problem
```
Multiple inheritance
Conflict resolution
Complexity
```

#### 4. Premature Abstraction
```
Wrong hierarchy chosen early
Hard to refactor later
Technical debt
```

### 5.2. Composition Advantages

**Benefits**:

#### 1. Flexibility
```
Mix and match behaviors
Runtime composition
No rigid hierarchy
```

#### 2. Reusability
```
Small, focused components
Combine freely
Share across projects
```

#### 3. Maintainability
```
Change one piece
Minimal impact
Easy to test
```

### 5.3. Composition Patterns

#### Pattern 1: Mixin
```
Merge objects
Add behavior
Multiple sources
```

#### Pattern 2: Trait
```
Named behaviors
Explicit composition
Conflict resolution
```

#### Pattern 3: Factory + Composition
```
Function returns composed object
Closure captures state
No classes needed
```

**Stampit library**: Composable factories

### 5.4. When to Use Each

**Use Inheritance when**:
- True "is-a" relationship
- Shallow hierarchy (< 3 levels)
- Unlikely to change
- Framework requirement

**Use Composition when**:
- "Has-a" or "uses-a" relationship
- Need flexibility
- Multiple behaviors
- Not sure about hierarchy

**Rule of thumb**: Prefer composition

---

## 6. Production Best Practices

### 6.1. Encapsulation Strategies

**Goals**:
- Hide implementation details
- Expose minimal API
- Prevent misuse

**Techniques**:

#### 1. Private Fields (#)
```
Pros: True privacy, syntax-level
Cons: Recent, needs transpilation
Use: Modern projects
```

#### 2. Closures
```
Pros: Works everywhere
Cons: Memory overhead
Use: When compatibility needed
```

#### 3. Naming Convention
```
Pros: Simple, no overhead
Cons: No enforcement
Use: Internal/team code
```

### 6.2. Error Handling

**Strategies**:

#### 1. Validation in Constructor
```
Fail fast
Clear errors
Prevent invalid state
```

#### 2. Defensive Programming
```
Check preconditions
Type validation
Throw descriptive errors
```

#### 3. Try-Catch Placement
```
At boundaries (API, UI)
Not everywhere
Let errors bubble
```

### 6.3. Testing OOP Code

**Challenges**:
- State management
- Dependencies
- Side effects

**Solutions**:

#### 1. Dependency Injection
```
Inject dependencies
Easy to mock
Clear contracts
```

#### 2. Interface-Based Design
```
Program to interfaces
Swap implementations
Test doubles easy
```

#### 3. Factory Functions
```
Create test instances
Control initialization
Isolation
```

### 6.4. Documentation

**What to document**:

#### Public API
```
JSDoc comments
Parameter types
Return types
Examples
```

#### Invariants
```
Class constraints
Assumptions
Preconditions
Postconditions
```

#### Design decisions
```
Why this pattern?
Trade-offs made
Alternatives considered
```

### 6.5. Performance Considerations

**Class creation**:
```
Cost: Minimal
Methods on prototype: Shared
Instance properties: Per object
```

**Inheritance depth**:
```
Shallow: Fast lookup
Deep: Slower
Recommendation: < 3 levels
```

**Private fields**:
```
#fields: Fast (optimized by engines)
WeakMap: Slower (extra lookup)
Closures: Memory overhead
```

---

## 📝 Key Takeaways

### Prototypal Inheritance
- **JavaScript's essence**: Objects inherit from objects
- **Prototype chain**: Lookup mechanism
- **Performance**: Shallow chains better
- **Prototype pollution**: Security concern

### Classes
- **Syntactic sugar**: Still prototypal underneath
- **Private fields (#)**: True privacy (ES2022)
- **Static members**: Class-level utilities
- **Inheritance**: Use sparingly, shallow hierarchies

### Design Patterns
- **Creational**: Singleton, Factory, Builder
- **Structural**: Adapter, Decorator, Proxy
- **Behavioral**: Observer, Strategy, Command, Module
- **Use wisely**: Patterns solve problems, not goals

### SOLID
- **SRP**: One responsibility per class
- **OCP**: Extend, don't modify
- **LSP**: Substitutability
- **ISP**: Small, focused interfaces
- **DIP**: Depend on abstractions

### Composition vs Inheritance
- **Inheritance Problems**: Fragile base class, gorilla-banana, diamond
- **Composition Benefits**: Flexibility, reusability, maintainability
- **When**: Prefer composition, use inheritance sparingly
- **Patterns**: Mixins, traits, factory + composition

### Production
- **Encapsulation**: Private fields, closures, conventions
- **Error Handling**: Fail fast, defensive programming, boundaries
- **Testing**: Dependency injection, interface-based, factories
- **Documentation**: Public API, invariants, design decisions
- **Performance**: Shallow hierarchies, method sharing, private field strategy

---

## 🎯 Advanced Topics

1. **Metaprogramming**: Proxies, Reflect API
2. **Mixins Libraries**: Stampit, traits
3. **Type Systems**: TypeScript classes, interfaces
4. **Framework Patterns**: React components, Vue components
5. **Architectural Patterns**: MVC, MVVM, Clean Architecture
6. **Domain-Driven Design**: Entities, Value Objects, Aggregates

---

**Chúc bạn học tốt! 🚀**
