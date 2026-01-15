# Advanced ES Features - Tính năng JavaScript nâng cao

> **Mục tiêu**: Nắm vững các tính năng ES6+ nâng cao, hiểu cơ chế hoạt động, và ứng dụng trong production.

---

## 📚 Table of Contents

1. [Proxy & Reflect - Metaprogramming](#1-proxy--reflect---metaprogramming)
2. [Symbols - Unique Identifiers](#2-symbols---unique-identifiers)
3. [Iterators & Generators - Custom Iteration](#3-iterators--generators---custom-iteration)
4. [Module Systems - ESM vs CommonJS](#4-module-systems---esm-vs-commonjs)
5. [Top-Level Await & Dynamic Imports](#5-top-level-await--dynamic-imports)
6. [Modern Language Features](#6-modern-language-features)

---

## 1. Proxy & Reflect - Metaprogramming

### 1.1. Tại Sao Cần Proxy?

**Problems solved**:

#### Problem 1: Property Access Control
```
Need: Validate, log, transform property access
Before: Object.defineProperty (limited)
Now: Proxy (13 traps)
```

#### Problem 2: Implementing DSLs
```
Need: Custom behavior for operators
Before: Hacks, workarounds
Now: Proxy traps
```

#### Problem 3: Reactive Systems
```
Need: Track dependencies automatically
Before: Manual tracking
Now: Proxy get/set traps (Vue 3)
```

### 1.2. Proxy Traps Overview

**13 fundamental traps**:

#### Object traps:
```
get, set → Property access
has → 'in' operator
deleteProperty → delete operator
ownKeys → Object.keys(), for...in
getOwnPropertyDescriptor → Object.getOwnPropertyDescriptor()
defineProperty → Object.defineProperty()
```

#### Prototype traps:
```
getPrototypeOf → Object.getPrototypeOf()
setPrototypeOf → Object.setPrototypeOf()
isExtensible → Object.isExtensible()
preventExtensions → Object.preventExtensions()
```

#### Function traps:
```
apply → Function call
construct → new operator
```

### 1.3. Real-World Use Cases

#### Use Case 1: Vue 3 Reactivity

**Mechanism**: Proxy-based reactive system

**How it works**:
```
1. Proxy wraps state object
2. get trap: Track dependencies
3. set trap: Trigger updates
4. Auto re-render components
```

**Benefits**:
- No API needed (direct property access)
- Deep reactivity
- Array methods work
- Better performance than Vue 2

#### Use Case 2: Validation Proxies

**Purpose**: Runtime type checking

**Applications**:
- Form validation
- API contracts
- Configuration objects
- Type guards

**Pattern**:
```
set trap checks type
Throw if invalid
Allow if valid
```

#### Use Case 3: Negative Array Indices

**Python-like behavior**: `arr[-1]` = last element

**Implementation**: get trap checks negative index

**Benefits**: More intuitive API

#### Use Case 4: Observable Objects

**Pattern**: Pub-Sub on object changes

**Use cases**:
- State management
- Data binding
- Change detection
- Undo/redo

#### Use Case 5: Access Control

**Purpose**: Security, encapsulation

**Techniques**:
```
Private prefix → Throw on access
Role-based → Check permissions
Read-only → Throw on set
```

### 1.4. Reflect API

**Purpose**: Default behavior for Proxy traps

**Why needed?**:
```
Without Reflect:
- Need to implement default manually
- Easy to get wrong
- Lose optimizations

With Reflect:
- Use Reflect.get/set/etc
- Correct behavior guaranteed
- Engine optimizations preserved
```

**Pattern**:
```javascript
new Proxy(target, {
  get(target, prop, receiver) {
    // Custom logic
    return Reflect.get(target, prop, receiver);  // Default behavior
  }
});
```

### 1.5. Performance Considerations

**Overhead**:
- Proxy slower than direct access (~10-20%)
- Hot paths: Avoid proxies
- Cold paths: Negligible impact

**Optimization strategies**:
- Cache proxy handlers
- Minimize trap logic
- Use Reflect for defaults
- Consider tradeoffs

**When acceptable**:
- Frameworks (Vue, MobX)
- Dev tools
- DSLs
- Not performance-critical paths

---

## 2. Symbols - Unique Identifiers

### 2.1. Why Symbols Exist

**Problems solved**:

#### Problem 1: Property Name Collisions
```
Before: String keys can clash
Now: Symbol keys are unique
Use: Library authors, frameworks
```

#### Problem 2: Hidden Properties
```
Before: Convention (_prefix)
Now: Symbol keys non-enumerable
Use: Metadata, internal state
```

#### Problem 3: Protocols
```
Before: Special string keys (__iterator__)
Now: Well-known symbols (Symbol.iterator)
Use: JavaScript built-ins
```

### 2.2. Symbol Types

#### Regular Symbols
```
Created: Symbol('description')
Unique: Always, even same description
Scope: Per creation
Use: Property keys, unique IDs
```

#### Global Symbols
```
Created: Symbol.for('key')
Registry: Global symbol registry
Same key: Same symbol across realms
Use: Cross-realm communication
```

### 2.3. Well-Known Symbols

**Purpose**: JavaScript protocol hooks

#### Symbol.iterator
```
Purpose: Make object iterable
Use: for...of, spread, destructuring
Implement: Return iterator
```

#### Symbol.toStringTag
```
Purpose: Customize Object.prototype.toString
Use: Better debugging, type info
Implement: Return string tag
```

#### Symbol.toPrimitive
```
Purpose: Custom type conversion
Hints: 'number', 'string', 'default'
Use: Custom numeric/string behavior
```

#### Symbol.hasInstance
```
Purpose: Customize instanceof
Use: Custom type checking
Implement: Return boolean
```

#### Others
```
Symbol.species → Constructor to use
Symbol.match → regex matching
Symbol.replace → string replace
Symbol.split → string split
Symbol.search → string search
```

### 2.4. Use Cases

#### Use Case 1: Private-ish Properties

**Pattern**: Symbol as property key

**Benefits**:
- Not enumerable
- Unique
- Hidden from Object.keys()

**Limitations**:
- Not truly private (Object.getOwnPropertySymbols)
- Use # for real privacy

#### Use Case 2: Singleton Pattern

**Pattern**: Symbol as registry key

**Benefits**: Cross-module uniqueness

#### Use Case 3: Custom Iterators

**Pattern**: Implement Symbol.iterator

**Use cases**:
- Custom collections
- Ranges
- Infinite sequences
- Tree traversals

#### Use Case 4: Framework Internals

**Examples**:
- React: Symbol for element type
- Redux: Symbol for$$observable
- Immer: Symbol for draft state

---

## 3. Iterators & Generators - Custom Iteration

### 3.1. Iterator Protocol

**Requirements**:
```
Object với next() method
Returns: { value, done }
done: true → iteration complete
```

**Use cases**:
- Custom collections
- Lazy sequences
- Streams
- Async data

### 3.2. Iterable Protocol

**Requirements**:
```
Object với [Symbol.iterator] method
Returns: Iterator
```

**Built-in iterables**:
- Arrays
- Strings
- Maps
- Sets
- NodeList

**Benefits of being iterable**:
- for...of works
- Spread operator
- Destructuring
- Array.from()

### 3.3. Generators Deep Dive

**Mechanism**:
```
function* → Generator function
yield → Pause and return
return → End generator
```

**States**:
```
Start → Suspended
yield → Suspended
next() → Executing
return/done → Completed
```

**Benefits**:
- Lazy evaluation
- Memory efficient
- Pausable execution
- Elegant syntax

### 3.4. Generator Use Cases

#### Use Case 1: Infinite Sequences

**Examples**:
- Fibonacci
- Prime numbers
- ID generation
- Timestamps

**Benefits**: Generate on demand, no memory limit

#### Use Case 2: Tree/Graph Traversal

**Algorithms**:
- BFS
- DFS
- Pre/in/post order

**Benefits**: Clean, readable implementation

#### Use Case 3: State Machines

**Pattern**: Each state as yield point

**Benefits**:
- Clear state transitions
- Pausable
- Testable

#### Use Case 4: Async Control Flow (Pre-async/await)

**Pattern**: Generator + runner (co, Redux-Saga)

**Benefits**:
- Synchronous-looking async code
- Cancellable
- Testable (yield effects)

**Status**: Mostly replaced by async/await

### 3.5. Async Generators

**Purpose**: Async iteration

**Syntax**:
```javascript
async function* name() {
  yield await promise;
}
```

**Consume with**: `for await...of`

**Use cases**:
- Streaming data
- Paginated APIs
- SSE (Server-Sent Events)
- File reading (chunks)

### 3.6. Generator Delegation

**Syntax**: `yield* iterable`

**Purpose**: Delegate to another generator/iterable

**Use cases**:
- Flatten nested structures
- Compose generators
- Recursive algorithms

---

## 4. Module Systems - ESM vs CommonJS

### 4.1. Evolution of Modules

#### Era 1: Global Scope (Pre-modules)
```
Problem: Global pollution, name conflicts
Solution: IIFE pattern
```

#### Era 2: AMD (RequireJS)
```
Target: Browsers
Loading: Async
Status: Legacy
```

#### Era 3: CommonJS (Node.js)
```
Target: Server
Loading: Sync
Status: Widely used
```

#### Era 4: ESM (Modern)
```
Target: Universal
Loading: Async (browsers), can be sync (Node)
Status: Standard
```

### 4.2. CommonJS Characteristics

**Syntax**:
```javascript
const module = require('./module');
module.exports = {...};
```

**Properties**:
- Dynamic: `require()` at runtime
- Synchronous loading
- Copies exports
- No tree-shaking
- Node.js default (historically)

**Benefits**:
- Simple mental model
- Dynamic imports easy
- Ecosystem mature

**Drawbacks**:
- No static analysis
- Bundle size larger
- Not browser-native

### 4.3. ESM Characteristics

**Syntax**:
```javascript
import module from './module.js';
export default {...};
```

**Properties**:
- Static: Declarations hoisted
- Asynchronous loading (browsers)
- Live bindings (references)
- Tree-shaking possible
- Browser native

**Benefits**:
- Static analysis
- Smaller bundles (tree-shaking)
- Future-proof
- Browser support

**Drawbacks**:
- Strict (must at top level, mostly)
- Learning curve
- Interop complexity

### 4.4. Key Differences

| Feature | CommonJS | ESM |
|---------|----------|-----|
| **Syntax** | `require/module.exports` | `import/export` |
| **Loading** | Sync | Async (browsers) |
| **Exports** | Copied value | Live binding |
| **Tree-shaking** | No | Yes |
| **Dynamic** | Yes | Limited (import()) |
| **Browser** | No (needs bundler) | Yes (native) |
| **File ext** | .js | .mjs or "type: module" |

### 4.5. Interoperability

**ESM importing CommonJS**:
```
Works in Node.js (default export)
Named exports via synthetic
```

**CommonJS importing ESM**:
```
Cannot require() ESM
Must use dynamic import()
Async only
```

**Best practice**: Use ESM for new code

### 4.6. Migration Strategy

**Steps**:
```
1. Use .mjs extension OR "type": "module"
2. Convert require → import
3. Convert module.exports → export
4. Handle dynamic requires (import())
5. Update build tools
6. Test thoroughly
```

**Gradual approach**: Dual package (CJS + ESM)

---

## 5. Top-Level Await & Dynamic Imports

### 5.1. Top-Level Await

**Before**: IIFE wrapper needed
```javascript
(async () => {
  const data = await fetch();
})();
```

**After**: Direct await
```javascript
const data = await fetch();
```

**Requirements**:
- ESM only
- Module becomes async
- Blocking for dependents

### 5.2. Use Cases for Top-Level Await

#### Use Case 1: Module Initialization

**Pattern**: Fetch config before module ready

**Benefits**:
- Clean syntax
- No IIFE
- Guaranteed ready

#### Use Case 2: Conditional Imports

**Pattern**: Await decision, then import

**Use case**: Feature flags, A/B tests

#### Use Case 3: Resource Initialization

**Examples**:
- Database connection
- i18n loading
- Environment config

### 5.3. Pitfalls of Top-Level Await

**Issue 1: Blocking**
```
Module awaits → Dependents wait
Long await → Slow startup
```

**Issue 2: Circular Dependencies**
```
Module A awaits B
Module B awaits A
→ Deadlock
```

**Best practices**:
- Use sparingly
- Fast operations only
- Document blocking
- Avoid circular deps

### 5.4. Dynamic Imports

**Syntax**: `import(modulePath)`

**Returns**: Promise resolving to module

**Benefits**:
- Code splitting
- Lazy loading
- Conditional loading
- Runtime paths

### 5.5. Dynamic Import Use Cases

#### Use Case 1: Route-Based Code Splitting

**Pattern**: Import route component on demand

**Benefits**:
- Smaller initial bundle
- Faster load time
- Better caching

**Frameworks**: React.lazy, Vue async components

#### Use Case 2: Feature Flags

**Pattern**: Import based on flag

**Benefits**:
- Conditional features
- A/B testing
- Progressive rollout

#### Use Case 3: Polyfills

**Pattern**: Import polyfill if needed

**Detection**: Feature detection

**Benefits**: Ship less to modern browsers

#### Use Case 4: Locale Loading

**Pattern**: Import locale on demand

**Benefits**: Smaller bundle, user-specific

### 5.6. Import.meta

**Purpose**: Module metadata

**Properties**:

#### import.meta.url
```
Current module URL
Resolve relative paths
File:// or http://
```

#### import.meta.env (Vite)
```
Environment variables
Build-time values
Type-safe (TS)
```

#### import.meta.hot (HMR)
```
Hot Module Replacement API
Accept updates
Dispose handlers
```

---

## 6. Modern Language Features

### 6.1. Optional Chaining (?.)

**Purpose**: Safe property access

**Variants**:
```
obj?.prop → Property
obj?.[expr] → Computed property
func?.(...args) → Function call
```

**Short-circuits**: Returns undefined if nullish

**Use cases**:
- API responses (optional fields)
- Nested objects
- Optional callbacks

**Gotcha**: `?.` checks only left side

### 6.2. Nullish Coalescing (??)

**Purpose**: Default values for null/undefined

**Difference from ||**:
```
|| → Falsy check (includes 0, '', false)
?? → Nullish check (only null, undefined)
```

**Use cases**:
- Configuration defaults
- Optional parameters
- API responses

**Combo**: `?.` + `??` for robust code

### 6.3. Logical Assignment

**Operators**: `&&=`, `||=`, `??=`

**Patterns**:

#### &&= (AND assignment)
```
x &&= y → x = x && y
Use: Update if truthy
```

#### ||= (OR assignment)
```
x ||= y → x = x || y
Use: Default value (falsy check)
```

#### ??= (Nullish assignment)
```
x ??= y → x = x ?? y
Use: Default value (nullish check)
```

### 6.4. Numeric Separators

**Purpose**: Readability in large numbers

**Syntax**:
```javascript
1_000_000 → 1000000
0xFF_FF_FF → 0xFFFFFF
0.000_001 → 0.000001
```

**Rules**: Cannot start/end, no consecutive

**Use**: Any numeric literal

### 6.5. BigInt

**Purpose**: Integers beyond Number.MAX_SAFE_INTEGER

**Syntax**: `123n` or `BigInt(123)`

**Operations**:
```
Arithmetic: +, -, *, **, %
Comparison: <, >, <=, >=, ==, ===
Bitwise: &, |, ^, ~, <<, >>
```

**Cannot mix**: BigInt and Number (TypeError)

**Use cases**:
- Timestamps (microseconds)
- Cryptography
- Large integers (IDs, counters)
- Financial calculations

### 6.6. Private Class Fields

**Syntax**: `#fieldName`

**Truly private**: SyntaxError outside class

**Use cases**:
- Encapsulation
- Internal state
- Implementation details

**Performance**: Optimized by engines

### 6.7. String Methods

#### replaceAll()
```
Before: regex with /g flag
Now: string.replaceAll(search, replace)
```

#### matchAll()
```
Purpose: All regex matches
Returns: Iterator
Benefits: Capture groups, lazy
```

#### at()
```
Purpose: Index access (negative supported)
string.at(-1) → last char
```

### 6.8. Promise Combinators

**Promise.allSettled()**:
```
Wait for all
Returns all results (fulfilled + rejected)
Use: Partial results needed
```

**Promise.any()**:
```
First fulfilled wins
All rejected → AggregateError
Use: Fastest response, fallbacks
```

**Comparison**:

| Method | Resolves | Rejects |
|--------|----------|---------|
| all | All fulfilled | Any rejected |
| race | First settled | First rejected |
| allSettled | Never (all done) | Never |
| any | First fulfilled | All rejected |

---

## 📝 Key Takeaways

### Proxy & Reflect
- **Metaprogramming**: 13 traps for customization
- **Use cases**: Vue 3 reactivity, validation, DSLs
- **Reflect**: Default behavior, use with Proxy
- **Performance**: 10-20% overhead, acceptable outside hot paths

### Symbols
- **Unique identifiers**: Avoid collisions
- **Well-known**: Symbol.iterator, toStringTag, etc.
- **Use cases**: Private properties, protocols, frameworks
- **Global registry**: Symbol.for() for cross-realm

### Iterators & Generators
- **Iterator**: next() → { value, done }
- **Generator**: function*, yield, pausable
- **Use cases**: Lazy sequences, traversals, state machines
- **Async generators**: for await...of, streaming

### Module Systems
- **CommonJS**: Sync, dynamic, Node.js legacy
- **ESM**: Static, tree-shakeable, future-proof
- **Differences**: Binding (copy vs live), analysis
- **Interop**: ESM can import CJS, not vice versa (sync)

### Top-Level Await & Dynamic Imports
- **Top-level await**: ESM only, blocks dependents
- **Use cases**: Config, initialization, conditional
- **Dynamic import()**: Code splitting, lazy loading
- **import.meta**: Module metadata (url, env, hot)

### Modern Features
- **Optional chaining (?.)**: Safe access
- **Nullish coalescing (??)**: null/undefined defaults
- **Logical assignment**: &&=, ||=, ??=
- **BigInt**: Arbitrary-precision integers
- **Private fields (#)**: True encapsulation
- **Promise combinators**: allSettled, any

---

## 🎯 Advanced Topics

1. **Realms API**: Isolated contexts
2. **Temporal API**: Modern date/time (Stage 3)
3. **Decorators**: Metadata, DI (Stage 3)
4. **Pattern Matching**: Switch improvements (Stage 1)
5. **Records & Tuples**: Immutable primitives (Stage 2)
6. **Pipeline Operator**: |> composition (Stage 2)

---

**Chúc bạn học tốt! 🚀**
