# Functional Programming - Lập trình hàm trong JavaScript

> **Mục tiêu**: Nắm vững FP concepts, patterns, và ứng dụng thực tế để viết code maintainable, testable, và scalable.

---

## 📚 Table of Contents

1. [Core Philosophy & Principles](#1-core-philosophy--principles)
2. [Pure Functions & Immutability](#2-pure-functions--immutability)
3. [Function Composition - The Power Tool](#3-function-composition---the-power-tool)
4. [Currying & Partial Application](#4-currying--partial-application)
5. [Functors & Monads - Practical Patterns](#5-functors--monads---practical-patterns)
6. [Real-World Applications](#6-real-world-applications)
7. [FP in Production](#7-fp-in-production)

---

## 1. Core Philosophy & Principles

### 1.1. Tại Sao Functional Programming?

**Vấn đề FP giải quyết**:

#### Problem 1: Side Effects - Uncontrolled State Mutations
```javascript
// ❌ Problematic: Hidden mutations, hard to reason
let totalPrice = 0;  // Global mutable state

function addĐến giỏHang(item) {
  totalPrice += item.price;  // Side effect!
  updateUI();  // Another side effect!
  logToAnalytics(item);  // Yet another!
  // Khó track: What else changes? In what order?
}

// Bugs when:
// - Multiple calls in wrong order
// - Concurrent execution
// - totalPrice modified elsewhere
```

**Impact**: 
- Hard to reason about program flow
- Hidden dependencies làm debugging nightmare
- Order-dependent execution → race conditions
- Cannot parallelize safely

#### Problem 2: Testing Complexity
```javascript
// ❌ Hard to test: Needs global state setup
class UserService {
  async getUser(id) {
    const db = getGlobalDB();  // Global dependency
    const cache = getGlobalCache();  // Another global
    // ... complex logic
  }
}

// Test requires:
// 1. Mock global DB
// 2. Mock global cache  
// 3. Setup/teardown overhead
// 4. Non-deterministic if state leaks
```

#### Problem 3: Concurrency Nightmares
```javascript
// ❌ Race condition with shared mutable state
let counter = 0;

// Thread 1:          Thread 2:
// read counter (0)   read counter (0)
// counter = 0 + 1    counter = 0 + 1
// write counter (1)  write counter (1)
// Result: counter = 1 (expected: 2!)

// Needs locks, mutexes → deadlock risks
```

**FP Solutions**:
- ✅ **Pure functions**: Predictable, no surprises - same input always same output
- ✅ **Immutability**: No race conditions - data never changes, only create new
- ✅ **Composition**: Modular, reusable - build complex from simple blocks
- ✅ **Declarative**: Express "what" not "how" - clearer intent

**Real-world Impact**:
- Facebook: React + Redux (immutable state) → easier debugging
- Elm: Zero runtime exceptions in production
- Haskell: Concurrency without locks

### 1.2. Fundamental Principles

#### 1. First-Class Functions
```
Functions là values
- Assign to variables
- Pass as arguments
- Return from functions
- Store in data structures
```

#### 2. Higher-Order Functions
```
Functions that:
- Take functions as arguments (map, filter)
- Return functions (factory, decorators)
```

#### 3. Pure Functions
```
Characteristics:
- Same input → Same output
- No side effects
- Referentially transparent
```

#### 4. Immutability
```
Data structures never change
Create new instead of mutate
Persistent data structures
```

#### 5. Function Composition
```
Build complex from simple
f(g(x)) pattern
Pipeline dataflow
```

#### 6. Declarative Programming
```
Express "what" not "how"
Abstraction over iteration
Intention over implementation
```

### 1.3. Benefits in Practice

**Predictability**:
- Pure functions → no surprises
- Immutable data → no hidden changes
- Explicit dependencies → clear contracts

**Testability**:
- No mocks needed (mostly)
- Isolated unit tests
- Property-based testing possible

**Composability**:
- Small, focused functions
- Easy to combine
- Reusable components

**Parallelizability**:
- No shared state → safe concurrency
- Map-reduce patterns
- Web Workers friendly

**Debugging**:
- Call stack preserved
- No hidden mutations
- Time-travel debugging possible

---

## 2. Pure Functions & Immutability

### 2.1. Pure Function Definition

**Requirements**:

1. **Deterministic**: Same inputs → Same outputs always
2. **No side effects**: No external dependencies/modifications

**NOT pure**:
- Reading/writing files
- Network requests
- DOM manipulation
- Modifying arguments
- console.log() technically!
- Math.random(), Date.now()

### 2.2. Benefits of Purity

**Memoization**:
```
Pure function → cacheable results
Same args seen before → return cached
Automatic optimization possible
```

**Parallelization**:
```
No shared state → parallel safe
Map across workers
GPU computing (WebGPU)
```

**Testing**:
```
No setup/teardown
Simple assertions
Property-based testing
```

**Reasoning**:
```
Local reasoning sufficient
No global context needed
Substitution model works
```

### 2.3. Immutability Strategies

#### Strategy 1: Shallow Copy
```
Spread operator: { ...obj }, [...arr]
Object.assign()
Array methods returning new: map, filter, slice
```

**Pros**: Simple, performant for small objects
**Cons**: Nested objects still mutable

#### Strategy 2: Deep Copy
```
JSON.parse(JSON.stringify()) - naive
structuredClone() - modern
Lodash cloneDeep() - robust
```

**Pros**: Complete immutability
**Cons**: Performance cost, some data types lost

#### Strategy 3: Structural Sharing
```
Libraries: Immutable.js, Immer
Share unchanged parts
Copy-on-write semantics
```

**Pros**: Performance + immutability
**Cons**: Library dependency, learning curve

#### Strategy 4: Persistent Data Structures
```
Concept: Versioned data structures
Each "mutation" creates new version
Share structure between versions
```
#### Decision Tree
```
Cần immutability?
├─ Object đơn giản, không nested?
│  → ✅ Spread operator
│
├─ Nested nhưng data nhỏ?
│  → ✅ structuredClone()
│
├─ Redux store hoặc form phức tạp?
│  → ✅ Immer (mutable syntax → immutable output)
│
└─ Frequent updates, data lớn?
   → ✅ Immutable.js (structural sharing)
```

**Examples**: Clojure vectors, Immutable.js

### 2.4. Immutability Patterns

#### Pattern 1: Update In Pattern
```
Philosophy: Functional updates deep in trees
Use case: Redux reducers, state management
Libraries: Ramda (over, set, lensProp)
```

#### Pattern 2: Lenses
```
Concept: Functional getters/setters
Composable property access
Immutable updates
```

#### Pattern 3: Immer Draft Pattern
```
Write mutable code
Library tracks changes
Produces immutable result
```

**Best of both worlds**: Mutable syntax, immutable semantics

### 2.5. Refactoring to Purity - Practical Techniques

**Mục tiêu**: Transform impure code → pure, testable functions

#### Impurity 1: Global State Access
```javascript
// ❌ Impure: Reads global config
function calculateDiscount(price) {
  const discountRate = window.config.discountRate;  // Global!
  return price * (1 - discountRate);
}

// ✅ Pure: Inject dependency
function calculateDiscount(price, discountRate) {
  return price * (1 - discountRate);
}
// Usage: calculateDiscount(100, window.config.discountRate)
```

#### Impurity 2: External I/O
```javascript
// ❌ Impure: Direct I/O
async function getUserData(id) {
  return await fetch(`/api/users/${id}`);  // Side effect!
}

// ✅ Pure: Return instruction (IO Monad pattern)
function getUserRequest(id) {
  return {
    type: 'FETCH',
    url: `/api/users/${id}`,
    method: 'GET'
  };  // Pure description, no execution
}
// Executor runs it separately:
const instruction = getUserRequest(123);
await executeIO(instruction);  // Impure part isolated
```

#### Impurity 3: Timing Dependencies
```javascript
// ❌ Impure: Current time
function logEvent(message) {
  const timestamp = Date.now();  // Non-deterministic!
  console.log(`[${timestamp}] ${message}`);
}

// ✅ Pure: Inject timestamp
function createLogEntry(message, timestamp) {
  return `[${timestamp}] ${message}`;
}
// Usage: createLogEntry('Event', Date.now())
```

#### Impurity 4: Randomness
```javascript
// ❌ Impure: Random generation
function generateId() {
  return Math.random().toString(36);  // Non-deterministic!
}

// ✅ Pure: Seed-based random
function generateId(seed) {
  // Seeded PRNG (pseudo-random number generator)
  const x = Math.sin(seed++) * 10000;
  return (x - Math.floor(x)).toString(36);
}
// Usage: generateId(seed++)  // Caller manages seed state
```

**Pattern**: **Push impurity to edges** - pure core, impure shell

---

## 3. Function Composition - The Power Tool

### 3.1. Composition Fundamentals

**Concept**: Build complex functions from simple ones

**Mathematical notation**: `(f ∘ g)(x) = f(g(x))`

**Direction**:
- **compose**: Right-to-left (mathematical)
- **pipe**: Left-to-right (intuitive)

### 3.2. Why Composition Matters

**Single Responsibility**:
```
Each function does one thing
Compose for complex behavior
Easy to test individual pieces
```

**Reusability**:
```
Small functions = building blocks
Combine in different ways
Domain-specific languages
```

**Refactoring**:
```
Extract common patterns
DRY principle naturally
Swap implementations easily
```

### 3.3. Composition Patterns

#### Pattern 1: Pipeline Pattern
```
Data flows through transformations
Each step processes and passes on
Like Unix pipes
```

**Use cases**:
- Data transformation (ETL)
- Request/response middleware
- Event processing

#### Pattern 2: Point-Free Style
```
Define functions without mentioning arguments
Composition of functions
Focus on transformations
```

**Benefits**: Less noise, clearer intent
**Drawbacks**: Can be cryptic if overused

#### Pattern 3: Transducers
```
Compose transformations efficiently
Single pass through data
No intermediate arrays
```

**Performance**: O(n) instead of O(n*m)

#### Pattern 4: Continuation Passing
```
Pass "what to do next" as argument
Chain async operations
Error handling built-in
```

### 3.4. Composition in Practice - Real-World Examples

#### Example 1: Data Processing Pipeline (Analytics)

**Scenario**: Process user analytics events from raw logs

**Implementation**:
```javascript
// Pure transformation functions
const parseJSON = (str) => JSON.parse(str);
const filterValidEvents = (event) => event.type && event.userId;
const enrichWithMetadata = (event) => ({
  ...event,
  timestamp: new Date(event.time),
  region: getUserRegion(event.userId)
});
const groupByUser = (events) => 
  events.reduce((acc, e) => {
    (acc[e.userId] = acc[e.userId] || []).push(e);
    return acc;
  }, {});

// Compose pipeline
const processEvents = pipe(
  map(parseJSON),           // Parse each line
  filter(filterValidEvents), // Keep valid only
  map(enrichWithMetadata),  // Add metadata
  groupByUser               // Group by user
);

// Execute
const rawLogs = ['{"type":"click",...}', ...];
const userEvents = processEvents(rawLogs);
```

**Benefits**: 
- Testable: Test each step independently
- Reusable: Mix/match transformations
- Clear: Intent obvious from pipeline

#### Example 2: Express Middleware Composition
```javascript
// Middleware = (req, res, next) => void
const authenticate = (req, res, next) => {
  if (!req.headers.authorization) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  req.user = verifyToken(req.headers.authorization);
  next();
};

const logRequest = (req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
};

const validateBody = (schema) => (req, res, next) => {
  const result = schema.validate(req.body);
  if (result.error) {
    return res.status(400).json({ error: result.error });
  }
  next();
};

// Compose middleware stack
app.post('/api/users', 
  logRequest,
  authenticate,
  validateBody(userSchema),
  createUserHandler
);

// Flow: Request → Log → Auth → Validate → Handler → Response
```

#### Example 3: React Higher-Order Components
```javascript
// HOC = Component → Enhanced Component
const withAuth = (Component) => (props) => {
  const { user, loading } = useAuth();
  if (loading) return <Loading />;
  if (!user) return <Redirect to="/login" />;
  return <Component {...props} user={user} />;
};

const withData = (fetcher) => (Component) => (props) => {
  const { data, loading, error } = useFetch(fetcher);
  if (loading) return <Loading />;
  if (error) return <Error error={error} />;
  return <Component {...props} data={data} />;
};

// Compose enhancers
const EnhancedUserProfile = compose(
  withAuth,
  withData(fetchUserProfile),
  withAnalytics
)(UserProfile);

// Equivalent to:
// withAuth(withData(withAnalytics(UserProfile)))
```

---

## 4. Currying & Partial Application

### 4.1. Currying Explained

**Definition**: Transform n-ary function → chain of unary functions

**Purpose**:
- Enable partial application
- Function specialization
- Point-free style

**When curried**:
```
All Ramda functions
Most Lodash/fp functions
Manual currying: curry() helper
```

### 4.2. Partial Application

**Definition**: Fix some arguments, create specialized function

**Use cases**:

#### Use Case 1: Configuration
```
Generic function + specific config = Specialized function
Example: fetchAPI(baseUrl, auth) → fetchUserAPI()
```

#### Use Case 2: Callback Specialization
```
Array.map needs unary function
Partial application adapts n-ary functions
```

#### Use Case 3: Dependency Injection
```
Inject dependencies early
Use later without re-specifying
```

### 4.3. Currying vs Partial Application

**Currying**:
- Transforms function signature
- Always returns unary functions
- Mathematical concept

**Partial Application**:
- Fixes arguments
- Returns function with remaining params
- Practical tool

**Both enable**: Function specialization and reuse

### 4.4. Advanced Patterns

#### Pattern 1: Auto-Currying
```
Functions curry automatically
Call with any number of args
Flexible arity
```

#### Pattern 2: Placeholder Arguments
```
_ placeholder for "fill this later"
Skip arguments, fill gaps
Ramda: R.__, Lodash: _
```

#### Pattern 3: Function Factories
```
Currying creates specialized functions
Factory pattern naturally
Configuration becomes first argument
```

---

## 5. Functors & Monads - Practical Patterns

### 5.1. Why Functors & Monads?

**Problem**: Handle effects functionally
- Null/undefined (Maybe/Option)
- Errors (Either/Result)
- Async (Promise/Task)
- State (State monad)
- Lists (Array)

**Solution**: Container types với functional operations

### 5.2. Functor Laws

**Definition**: Type với map operation

**Laws**:
```javascript
// Identity
map(x => x) ≡ identity

// Composition
map(f).map(g) ≡ map(x => g(f(x)))
```

**Built-in Functors**: Array, Promise

### 5.3. Maybe/Option Pattern

**Problem**: Null pointer errors

**Solution**: Explicit optional type

**Operations**:
- `map`: Transform value if present
- `chain`/`flatMap`: Avoid nesting
- `getOrElse`: Provide default

**Use cases**:
- Chained property access
- Optional configuration
- Fallback values

**Benefits**:
- No null checks scattered
- Explicit optionality
- Pipeline-friendly

### 5.4. Either/Result Pattern

**Problem**: Error handling với exceptions

**Solution**: Explicit success/failure type

**Variants**:
- `Left`: Error/failure case
- `Right`: Success case

**Operations**:
- `map`: Only on Right
- `mapLeft`: Only on Left
- `fold`: Handle both cases

**Use cases**:
- Validation
- Parsing
- API responses

**Benefits**:
- Type-safe errors
- No exceptions
- Composable error handling

### 5.5. Promise as Monad

**Monad operation**: `then` (flatMap)

**Laws satisfied**:
- Left identity
- Right identity
- Associativity

**Practical implications**:
- Promise chaining works compositionally
- Error propagation automatic
- Async composition clean

### 5.6. Task Pattern (Lazy Promise)

**Problem**: Promise executes immediately

**Solution**: Task - lazy async computation

**Benefits**:
- Control execution timing
- Retry-able
- Cancel-able
- Composable before execution

**Use cases**:
- API request builders
- Test mocks
- Complex async workflows

---

## 6. Real-World Applications

### 6.1. Redux & State Management

**FP Principles**:
- **Pure reducers**: (state, action) → newState
- **Immutable state**: Update patterns
- **Selectors**: Pure computed values
- **Action creators**: Function factories

**Benefits**:
- Predictable state changes
- Time-travel debugging
- Easy testing
- DevTools integration

### 6.2. React Hooks

**FP Concepts**:
- `useState`: State monad-ish
- `useEffect`: Side effect management
- `useCallback`: Memoized functions
- `useMemo`: Lazy evaluation

**Custom hooks**: Higher-order functions for reuse

### 6.3. Form Validation

**FP Approach**:

**Validators**: Pure predicates
```
email, minLength, required → Boolean
```

**Composition**:
```
Combine validators
all, any, none patterns
```

**Errors**: Either/Result monad
```
Success: Valid data
Failure: Validation errors
```

**Benefits**: Reusable, testable, declarative

### 6.4. API Client Design

**Layered composition**:

**Layer 1**: HTTP client (fetch)
**Layer 2**: Auth middleware
**Layer 3**: Error handling
**Layer 4**: Retries
**Layer 5**: Caching

**Each layer**: Pure transformation hoặc effect wrapper

### 6.5. Data Transformation Pipelines

**ETL processes**:
```
Extract → Transform → Load
Each step: Pure function
Compose into pipeline
```

**Examples**:
- Log processing
- Analytics aggregation
- Report generation

---

## 7. FP in Production

### 7.1. When to Use FP

**Good fit**:
- Data transformations
- Business logic
- Validation
- State management
- Testing

**Less suitable**:
- Performance-critical hot paths
- Heavy mutation (e.g., game loops)
- Highly stateful (e.g., canvas drawing)

### 7.2. Library Choices

#### Ramda
```
Pros:
- Full-featured FP library
- Auto-curried functions
- Great documentation

Cons:
- Bundle size (~50KB)
- Learning curve

When: Full FP adoption
```

#### Lodash/fp
```
Pros:
- Familiar Lodash API
- Smaller (~20KB)
- Gradual adoption easy

Cons:
- Not as pure
- Some non-FP utilities remain

When: Incremental FP
```

#### Folktale
```
Pros:
- Algebraic types (Maybe, Result, Task)
- Monadic patterns

Cons:
- Smaller ecosystem
- Less mainstream

When: Need monads
```

### 7.3. Performance Considerations

**Potential issues**:

#### Issue 1: Allocation Overhead
```
Problem: New objects on every update
Impact: GC pressure
Solution: Structural sharing libraries
```

#### Issue 2: Call Stack
```
Problem: Deep composition chains
Impact: Stack overflow risk
Solution: Trampolining, iterative
```

#### Issue 3: Bundle Size
```
Problem: FP libraries add KBs
Impact: Load time
Solution: Tree-shaking, pick minimal functions
```

**Optimization strategies**:
- Memoization for expensive pure functions
- Transducers for multi-step transformations
- Lazy evaluation where possible

### 7.4. Team Adoption

**Challenges**:
- Learning curve
- Unfamiliar syntax
- Debugging complexity

**Strategies**:

#### 1. Incremental Adoption
```
Start: Pure utility functions
Then: Immutable data patterns
Finally: Full FP with monads
```

#### 2. Documentation
```
Explain "why" not just "how"
Use familiar examples
Show benefits concretely
```

#### 3. Pair Programming
```
Share FP knowledge
Review each other's code
Build shared understanding
```

### 7.5. Testing FP Code

**Advantages**:
- Pure functions = simple tests
- No mocks needed (mostly)
- Property-based testing

**Property-Based Testing**:
```
Instead of example-based tests
Define properties that should hold
Generate random inputs
Verify properties
```

**Tools**: fast-check, JSVerify

---

## 📝 Key Takeaways

### Core Principles
- **Pure functions**: Predictable, testable, composable
- **Immutability**: No hidden changes, safe concurrency
- **First-class functions**: Functions as data
- **Composition**: Build complex from simple

### Pure Functions & Immutability
- Benefits: Memoization, parallelization, testing
- Strategies: Shallow copy, deep copy, structural sharing
- Patterns: Lenses, Immer, persistent data structures

### Function Composition
- **Pipe**: Left-to-right dataflow
- **Point-free**: Argument-less function definitions
- **Transducers**: Efficient multi-step transformations
- Use cases: Data pipelines, middleware, HOCs

### Currying & Partial Application
- **Currying**: n-ary → chain of unary
- **Partial**: Fix arguments, specialize
- Use cases: Configuration, dependency injection, callbacks

### Functors & Monads
- **Maybe**: Handle optional values
- **Either**: Error handling without exceptions
- **Promise**: Async monad
- **Task**: Lazy async
- Practical patterns for effects

### Real-World Applications
- Redux: Pure reducers, immutable state
- React Hooks: FP patterns in React
- Form validation: Composable validators
- API clients: Layered composition
- ETL: Transformation pipelines

### Production
- Choose library wisely (Ramda, Lodash/fp)
- Performance: Watch allocations, stack, bundle size
- Incremental adoption strategy
- Property-based testing
- Team education important

---

## 🎯 Advanced Topics

1. **Category Theory**: Understand the math
2. **Free Monads**: Interpret programs
3. **Reactive Programming**: RxJS, FRP
4. **Lens Libraries**: Ramda, Monocle
5. **Type-Level Programming**: TypeScript FP
6. **Algebraic Effects**: Future of effects

---

**Chúc bạn học tốt! 🚀**
