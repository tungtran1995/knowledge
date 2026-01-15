# State Machines - Concept Guide

> **Core idea:** Make impossible states impossible. UI can only be in ONE valid state at a time.

---

## 1. The Problem with Booleans

### Boolean Hell

```javascript
// ❌ Multiple booleans = Impossible states possible
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);

// What if all are true? Which to show?
if (isLoading && isError && isSuccess) {
  return "???"; // Impossible state!
}
```

**Problem:** 3 booleans = 8 possible combinations (2³), but only 4 are valid.

---

### State Machine Solution

```javascript
// ✅ Only one state at a time
const [state, setState] = useState('idle');

// States: 'idle' | 'loading' | 'success' | 'error'
// Impossible to be loading AND error simultaneously
```

**Benefit:** Impossible states eliminated by design.

---

## 2. What is a State Machine?

### Definition

```
State Machine = System that can be in ONE of a finite number of states

Components:
1. States (idle, loading, success, error)
2. Events (FETCH, SUCCESS, ERROR)
3. Transitions (loading --SUCCESS--> success)
```

---

### Mental Model: Traffic Light

```
States: Red | Yellow | Green

Transitions:
Red --TIMER--> Green
Green --TIMER--> Yellow
Yellow --TIMER--> Red

Cannot be: Red AND Green (impossible)
```

---

## 3. Common UI Patterns

### Pattern 1: Data Fetching

**States:**
```
idle → User hasn't clicked yet
loading → Fetching data
success → Data loaded
error → Something failed
```

**Implementation:**

```javascript
// Simple (useState)
function DataFetcher() {
  const [state, setState] = useState('idle');
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  
  const fetch = async () => {
    setState('loading');
    
    try {
      const result = await api.getData();
      setData(result);
      setState('success');
    } catch (err) {
      setError(err);
      setState('error');
    }
  };
  
  return (
    <div>
      {state === 'idle' && <button onClick={fetch}>Load</button>}
      {state === 'loading' && <Spinner />}
      {state === 'success' && <Data data={data} />}
      {state === 'error' && <Error error={error} retry={fetch} />}
    </div>
  );
}
```

---

### Pattern 2: Form Submission

**States:**
```
editing → User typing
validating → Checking input
submitting → Sending to server
success → Submitted successfully
error → Submission failed
```

**Transitions:**
```
editing --SUBMIT--> validating
validating --VALID--> submitting
validating --INVALID--> editing
submitting --SUCCESS--> success
submitting --ERROR--> error
error --RETRY--> submitting
```

---

### Pattern 3: Multi-Step Wizard

**States:**
```
step1 → Personal info
step2 → Address
step3 → Payment
step4 → Review
complete → Order placed
```

**Transitions:**
```
step1 --NEXT--> step2
step2 --NEXT--> step3
step2 --BACK--> step1
step3 --SUBMIT--> complete
```

---

## 4. XState Library (Industry Standard)

### Basic Usage

```javascript
import { createMachine, useMachine } from '@xstate/react';

// Define machine
const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  states: {
    idle: {
      on: { FETCH: 'loading' }
    },
    loading: {
      on: {
        SUCCESS: 'success',
        ERROR: 'error'
      }
    },
    success: {
      on: { REFETCH: 'loading' }
    },
    error: {
      on: { RETRY: 'loading' }
    }
  }
});

// Use in component
function DataFetcher() {
  const [state, send] = useMachine(fetchMachine);
  
  const fetch = async () => {
    send('FETCH');
    
    try {
      const data = await api.getData();
      send('SUCCESS', { data });
    } catch (error) {
      send('ERROR', { error });
    }
  };
  
  return (
    <div>
      {state.matches('idle') && <button onClick={fetch}>Load</button>}
      {state.matches('loading') && <Spinner />}
      {state.matches('success') && <Data />}
      {state.matches('error') && <Error retry={() => send('RETRY')} />}
    </div>
  );
}
```

---

### With Context (Data Storage)

```javascript
const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  context: {
    data: null,
    error: null
  },
  states: {
    idle: {
      on: { FETCH: 'loading' }
    },
    loading: {
      invoke: {
        src: 'fetchData',
        onDone: {
          target: 'success',
          actions: assign({
            data: (context, event) => event.data
          })
        },
        onError: {
          target: 'error',
          actions: assign({
            error: (context, event) => event.data
          })
        }
      }
    },
    success: {},
    error: {
      on: { RETRY: 'loading' }
    }
  }
}, {
  services: {
    fetchData: () => api.getData()
  }
});
```

---

## 5. When to Use State Machines

### ✅ Good Fit

**Complex UI states:**
- Multi-step forms
- Authentication flows (login → MFA → success)
- Wizards / onboarding
- Game states

**Race conditions:**
- Multiple async operations
- User can trigger events rapidly
- Need prevent invalid transitions

**Team alignment:**
- Visual diagrams (everyone understands flow)
- Clear contracts (states & events documented)

---

### ❌ Overkill

**Simple cases:**
```javascript
// Don't need state machine for simple toggle
const [isOpen, setIsOpen] = useState(false);
```

**Static data:**
```javascript
// Don't need for data that doesn't change states
const config = { theme: 'dark', lang: 'en' };
```

**Rule:** If you have 2-3 states, useState sufficient. If 4+ states with transitions, consider state machine.

---

## 6. Benefits

### 1. **Impossible States Prevented**

```javascript
// ❌ Boolean hell: Can be loading AND error
setIsLoading(true);
setIsError(true);

// ✅ State machine: Cannot be loading AND error
state = 'loading'; // OR
state = 'error';   // Never both
```

---

### 2. **Visual Documentation**

```
[State Machine Diagram]

     FETCH
idle -----> loading
              |
      SUCCESS | ERROR
              |
         +----+----+
         |         |
      success    error
                   |
              RETRY |
                   |
                loading
```

**Everyone understands flow** → Better team collaboration.

---

### 3. **Predictable Behavior**

```javascript
// State machine enforces valid transitions
if (currentState === 'success') {
  // Can only go to 'loading' (refetch)
  // Cannot go to 'idle' or 'error'
}
```

---

### 4. **Easy Testing**

```javascript
test('loading -> success transition', () => {
  const machine = interpret(fetchMachine);
  
  machine.start();
  expect(machine.state.value).toBe('idle');
  
  machine.send('FETCH');
  expect(machine.state.value).toBe('loading');
  
  machine.send('SUCCESS');
  expect(machine.state.value).toBe('success');
});
```

---

## 7. State Machine vs React Query

### When to Use Which?

**React Query (Data Fetching):**
```javascript
const { data, isLoading, error } = useQuery('posts', fetchPosts);

// Good for: Simple data fetching, caching, automatic refetch
```

**State Machine (Complex Logic):**
```javascript
const [state, send] = useMachine(authMachine);

// Good for: Multi-step flows, complex transitions, business logic
```

---

**Hybrid Approach:**
```javascript
// React Query for data
const { data } = useQuery('user', fetchUser);

// State machine for flow
const [state, send] = useMachine(checkoutMachine);

// Best of both worlds
```

---

## 8. Common Patterns

### Pattern 1: Auth Flow

```javascript
const authMachine = createMachine({
  initial: 'loggedOut',
  states: {
    loggedOut: {
      on: { LOGIN: 'authenticating' }
    },
    authenticating: {
      on: {
        SUCCESS: 'checkingMFA',
        ERROR: 'loggedOut'
      }
    },
    checkingMFA: {
      on: {
        REQUIRED: 'mfaPrompt',
        NOT_REQUIRED: 'loggedIn'
      }
    },
    mfaPrompt: {
      on: {
        SUBMIT: 'verifyingMFA',
        CANCEL: 'loggedOut'
      }
    },
    verifyingMFA: {
      on: {
        SUCCESS: 'loggedIn',
        ERROR: 'mfaPrompt'
      }
    },
    loggedIn: {
      on: { LOGOUT: 'loggedOut' }
    }
  }
});
```

---

### Pattern 2: Payment Flow

```javascript
const paymentMachine = createMachine({
  initial: 'selectingMethod',
  states: {
    selectingMethod: {
      on: {
        SELECT_CARD: 'enteringCard',
        SELECT_PAYPAL: 'paypal'
      }
    },
    enteringCard: {
      on: {
        SUBMIT: 'validating',
        BACK: 'selectingMethod'
      }
    },
    validating: {
      on: {
        VALID: 'processing',
        INVALID: 'enteringCard'
      }
    },
    processing: {
      on: {
        SUCCESS: 'complete',
        ERROR: 'failed'
      }
    },
    failed: {
      on: { RETRY: 'processing' }
    },
    complete: { type: 'final' }
  }
});
```

---

## 9. XState DevTools

### Visualizer

```javascript
import { inspect } from '@xstate/inspect';

// Development only
if (process.env.NODE_ENV === 'development') {
  inspect({ iframe: false });
}

const [state, send] = useMachine(machine, { devTools: true });
```

**Features:**
- Visual state diagram
- Current state highlighted
- Event history
- Time-travel debugging

**URL:** https://stately.ai/viz (paste machine config)

---

## 10. Migration from Booleans

### Before (Boolean Hell)

```javascript
const [isIdle, setIsIdle] = useState(true);
const [isLoading, setIsLoading] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [isError, setIsError] = useState(false);

const handleFetch = async () => {
  setIsIdle(false);
  setIsLoading(true);
  
  try {
    const data = await api.fetch();
    setIsLoading(false);
    setIsSuccess(true);
  } catch (error) {
    setIsLoading(false);
    setIsError(true);
  }
};
```

---

### After (State Machine)

```javascript
const [state, send] = useMachine(fetchMachine);

const handleFetch = () => {
  send('FETCH');
};

// Machine handles all state transitions internally
```

**Improvement:** 16 lines → 5 lines, impossible states eliminated.

---

## 11. Tổng Kết

### Core Concepts

```
1. Finite states (only ONE at a time)
2. Explicit transitions (state A -> event -> state B)
3. Impossible states prevented
4. Predictable behavior
```

---

### When to Use

```
✅ Use state machines:
- 4+ states with complex transitions
- Multi-step flows (forms, checkout, auth)
- Race conditions need prevention
- Team needs visual documentation

❌ Don't use:
- Simple toggles (isOpen)
- 2-3 states
- Static data
```

---

### Quick Decision

```
Ask: "Can this be in multiple conflicting states simultaneously?"

If YES → State machine
If NO → useState sufficient
```

---

### Key Insight

> **Good architecture = Making invalid states unrepresentable.**

State machines achieve this by design.

**Action:** Next time you have 3+ booleans, consider refactoring to state machine.

---

**Further Learning:**
- XState Docs: https://xstate.js.org
- Visualizer: https://stately.ai/viz
- Examples: https://xstate.js.org/docs/examples