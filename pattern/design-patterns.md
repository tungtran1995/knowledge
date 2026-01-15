# Design Patterns for Frontend

> **Adapted from Gang of Four patterns for React/Frontend context.**

---

## 1. Creational Patterns

### Singleton Pattern

**Concept:** Ensure only one instance exists.

**Use cases:**
- Global store (Redux store)
- API client
- Logger
- Configuration

**Implementation:**

```javascript
// ES6 Module = natural singleton
// api-client.js
class APIClient {
  constructor() {
    this.baseURL = process.env.API_URL;
  }
  
  get(url) { /* ... */ }
  post(url, data) { /* ... */ }
}

// Only created once
export default new APIClient();

// Usage (always same instance)
import apiClient from './api-client';
apiClient.get('/users');
```

**React Context (Singleton-like):**
```javascript
// One provider at root = singleton behavior
<AuthProvider>
  <App />
</AuthProvider>
```

---

### Factory Pattern

**Concept:** Create objects without specifying exact class.

**Use case:** Create components dynamically based on type.

```javascript
// Component factory
const componentFactory = {
  text: TextInput,
  number: NumberInput,
  email: EmailInput,
  select: SelectInput
};

function FormField({ type, ...props }) {
  const Component = componentFactory[type];
  
  if (!Component) {
    throw new Error(`Unknown field type: ${type}`);
  }
  
  return <Component {...props} />;
}

// Usage
<FormField type="email" label="Email" />
<FormField type="select" label="Country" options={countries} />
```

---

### Builder Pattern

**Concept:** Construct complex objects step by step.

**Use case:** Complex API query builder.

```javascript
class QueryBuilder {
  constructor() {
    this.filters = [];
    this.sorts = [];
    this.pageSize = 20;
  }
  
  where(field, operator, value) {
    this.filters.push({ field, operator, value });
    return this; // Chainable
  }
  
  orderBy(field, direction = 'asc') {
    this.sorts.push({ field, direction });
    return this;
  }
  
  limit(size) {
    this.pageSize = size;
    return this;
  }
  
  build() {
    return {
      filters: this.filters,
      sorts: this.sorts,
      pageSize: this.pageSize
    };
  }
}

// Usage
const query = new QueryBuilder()
  .where('status', '=', 'active')
  .where('age', '>', 18)
  .orderBy('createdAt', 'desc')
  .limit(50)
  .build();
```

---

## 2. Structural Patterns

### Adapter Pattern

**Concept:** Convert one interface to another.

**Use case:** Adapt third-party API to your app's interface.

```javascript
// Third-party API
class LegacyUserAPI {
  getUserData(userId) {
    return {
      user_id: userId,
      full_name: 'John Doe',
      email_address: 'john@example.com'
    };
  }
}

// Adapter
class UserAPIAdapter {
  constructor(legacyAPI) {
    this.legacyAPI = legacyAPI;
  }
  
  getUser(id) {
    const data = this.legacyAPI.getUserData(id);
    
    // Adapt to our format
    return {
      id: data.user_id,
      name: data.full_name,
      email: data.email_address
    };
  }
}

// Usage
const legacyAPI = new LegacyUserAPI();
const userAPI = new UserAPIAdapter(legacyAPI);
const user = userAPI.getUser(123); // { id, name, email }
```

---

### Decorator Pattern

**Concept:** Add behavior to objects dynamically.

**Use case:** Higher-Order Components (HOCs) in React.

```javascript
// Decorator: Add loading state
function withLoading(Component) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) {
      return <Spinner />;
    }
    return <Component {...props} />;
  };
}

// Decorator: Add error boundary
function withErrorBoundary(Component) {
  return class extends React.Component {
    state = { hasError: false };
    
    static getDerivedStateFromError() {
      return { hasError: true };
    }
    
    render() {
      if (this.state.hasError) {
        return <ErrorFallback />;
      }
      return <Component {...this.props} />;
    }
  };
}

// Compose decorators
const EnhancedComponent = withErrorBoundary(
  withLoading(UserProfile)
);
```

---

### Proxy Pattern

**Concept:** Control access to an object.

**Use case:** API caching proxy.

```javascript
class APIProxy {
  constructor(api) {
    this.api = api;
    this.cache = new Map();
  }
  
  async get(url) {
    // Check cache
    if (this.cache.has(url)) {
      console.log('Cache hit');
      return this.cache.get(url);
    }
    
    // Fetch from API
    console.log('Cache miss, fetching...');
    const data = await this.api.get(url);
    
    // Store in cache
    this.cache.set(url, data);
    
    return data;
  }
}

// Usage
const api = new APIClient();
const cachedAPI = new APIProxy(api);

await cachedAPI.get('/users'); // Fetch
await cachedAPI.get('/users'); // Cache hit
```

---

### Facade Pattern

**Concept:** Simplified interface to complex system.

**Use case:** Simplify complex library usage.

```javascript
// Complex libraries
import { encode } from 'library-a';
import { compress } from 'library-b';
import { upload } from 'library-c';

// Facade
class FileUploader {
  async uploadFile(file) {
    // Hide complexity
    const encoded = encode(file);
    const compressed = compress(encoded);
    const result = await upload(compressed);
    return result;
  }
}

// Simple usage
const uploader = new FileUploader();
await uploader.uploadFile(myFile);
```

---

## 3. Behavioral Patterns

### Observer Pattern

**Concept:** Notify subscribers when state changes.

**Use case:** Event system, Pub/Sub.

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }
  
  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
  }
  
  emit(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(callback => callback(data));
    }
  }
  
  off(event, callback) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(cb => cb !== callback);
    }
  }
}

// Usage
const emitter = new EventEmitter();

emitter.on('user-login', (user) => {
  console.log('User logged in:', user);
});

emitter.on('user-login', (user) => {
  trackEvent('login', user);
});

emitter.emit('user-login', { id: 1, name: 'John' });
```

**React:** `useState`, `useEffect` = built-in observer pattern.

---

### Strategy Pattern

**Concept:** Select algorithm at runtime.

**Use case:** Different payment methods, sort strategies.

```javascript
// Strategies
const paymentStrategies = {
  creditCard: (amount) => {
    console.log(`Processing credit card payment: $${amount}`);
    // Stripe logic
  },
  
  paypal: (amount) => {
    console.log(`Processing PayPal payment: $${amount}`);
    // PayPal SDK
  },
  
  crypto: (amount) => {
    console.log(`Processing crypto payment: $${amount}`);
    // Blockchain logic
  }
};

// Context
class PaymentProcessor {
  processPayment(method, amount) {
    const strategy = paymentStrategies[method];
    
    if (!strategy) {
      throw new Error(`Unknown payment method: ${method}`);
    }
    
    strategy(amount);
  }
}

// Usage
const processor = new PaymentProcessor();
processor.processPayment('creditCard', 100);
processor.processPayment('paypal', 50);
```

---

### Command Pattern

**Concept:** Encapsulate request as object.

**Use case:** Undo/redo functionality.

```javascript
// Commands
class AddTodoCommand {
  constructor(todos, todo) {
    this.todos = todos;
    this.todo = todo;
  }
  
  execute() {
    this.todos.push(this.todo);
  }
  
  undo() {
    this.todos.pop();
  }
}

class DeleteTodoCommand {
  constructor(todos, index) {
    this.todos = todos;
    this.index = index;
    this.deletedTodo = null;
  }
  
  execute() {
    this.deletedTodo = this.todos.splice(this.index, 1)[0];
  }
  
  undo() {
    this.todos.splice(this.index, 0, this.deletedTodo);
  }
}

// Invoker
class CommandHistory {
  constructor() {
    this.history = [];
    this.currentIndex = -1;
  }
  
  execute(command) {
    command.execute();
    this.history = this.history.slice(0, this.currentIndex + 1);
    this.history.push(command);
    this.currentIndex++;
  }
  
  undo() {
    if (this.currentIndex >= 0) {
      this.history[this.currentIndex].undo();
      this.currentIndex--;
    }
  }
  
  redo() {
    if (this.currentIndex < this.history.length - 1) {
      this.currentIndex++;
      this.history[this.currentIndex].execute();
    }
  }
}

// Usage
const todos = [];
const history = new CommandHistory();

history.execute(new AddTodoCommand(todos, 'Task 1'));
history.execute(new AddTodoCommand(todos, 'Task 2'));
history.undo(); // Remove Task 2
history.redo(); // Add Task 2 back
```

---

### Template Method Pattern

**Concept:** Define skeleton, subclasses fill in details.

**Use case:** Form validation with different rules.

```javascript
class FormValidator {
  validate(data) {
    // Template method
    this.validateRequired(data);
    this.validateFormat(data);
    this.validateBusinessRules(data);
    return this.errors;
  }
  
  validateRequired(data) {
    // Default implementation
  }
  
  validateFormat(data) {
    // Override in subclass
    throw new Error('Must implement validateFormat');
  }
  
  validateBusinessRules(data) {
    // Override in subclass
  }
}

class EmailFormValidator extends FormValidator {
  validateFormat(data) {
    if (!data.email.includes('@')) {
      this.errors.push('Invalid email format');
    }
  }
  
  validateBusinessRules(data) {
    if (data.email.endsWith('@competitor.com')) {
      this.errors.push('Competitor email not allowed');
    }
  }
}
```

---

## 4. React-Specific Patterns

### Provider Pattern

**Concept:** Share data via Context.

```javascript
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('dark');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Usage
<ThemeProvider>
  <App />
</ThemeProvider>
```

---

### Container/Presentational Pattern

**Concept:** Separate logic from UI.

```javascript
// Container (logic)
function UserListContainer() {
  const { data, loading } = useQuery('users', fetchUsers);
  
  if (loading) return <Spinner />;
  
  return <UserList users={data} />;
}

// Presentational (UI only)
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

---

### Hooks Pattern

**Concept:** Reusable stateful logic.

```javascript
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });
  
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);
  
  return [value, setValue];
}

// Usage
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'dark');
}
```

---

## Tổng Kết

**When to use patterns:**

✅ **Use when:**
- Solving recurring problems
- Need proven solutions
- Team needs common language

❌ **Don't use when:**
- Over-engineering simple problems
- Pattern doesn't fit
- Team doesn't understand pattern

**Modern Frontend = Mix of:**
- Traditional patterns (Singleton, Factory, Observer)
- React patterns (Hooks, Provider, Compound Components)
- Custom patterns for your domain

**Key:** Know patterns, but don't force them. Use when they clarify code, not complicate it.

---

**References:**
- Gang of Four: Design Patterns book
- patterns.dev (React-specific)
- refactoring.guru (visual explanations)

Ah hiểu rồi! Vậy bạn đang tách riêng **General FE Patterns** vs **React-specific Patterns**. Đúng hướng!

Vậy tôi sẽ review lại và đề xuất bổ sung cho **General FE Patterns** thôi (không include React-specific):

---

## 📊 Review lại file hiện tại

Bạn đã có:
- ✅ Creational: Singleton, Factory, Builder
- ✅ Structural: Adapter, Decorator, Proxy, Facade  
- ✅ Behavioral: Observer, Strategy, Command, Template Method

**Đây là foundation tốt, nhưng thiếu những pattern quan trọng khác trong GOF và FE modern.**

---

## 🔥 Patterns còn thiếu trong GOF (nên có)

### 1. **Prototype Pattern** (Creational)
Quan trọng với object cloning, deep copy/shallow copy

```javascript
class UserProfile {
  constructor(data) {
    this.name = data.name;
    this.settings = data.settings;
  }
  
  clone() {
    // Deep clone
    return new UserProfile({
      name: this.name,
      settings: { ...this.settings }
    });
  }
}

// Usage
const user1 = new UserProfile({ 
  name: 'John', 
  settings: { theme: 'dark' } 
});
const user2 = user1.clone();
user2.settings.theme = 'light'; // không ảnh hưởng user1
```

**Use cases FE:**
- Clone form state để undo
- Copy configuration objects
- Template cloning (email templates, dashboard layouts)

---

### 2. **Composite Pattern** (Structural)
Cực kỳ quan trọng cho tree structures

```javascript
// Component interface
class UIComponent {
  render() {
    throw new Error('Must implement render');
  }
}

// Leaf
class Button extends UIComponent {
  constructor(label) {
    super();
    this.label = label;
  }
  
  render() {
    return `<button>${this.label}</button>`;
  }
}

// Composite
class Panel extends UIComponent {
  constructor() {
    super();
    this.children = [];
  }
  
  add(component) {
    this.children.push(component);
  }
  
  render() {
    const content = this.children
      .map(child => child.render())
      .join('');
    return `<div class="panel">${content}</div>`;
  }
}

// Usage - treat single & composite uniformly
const panel = new Panel();
panel.add(new Button('Save'));
panel.add(new Button('Cancel'));

const mainPanel = new Panel();
mainPanel.add(panel); // Nested!
mainPanel.add(new Button('Submit'));

console.log(mainPanel.render());
```

**Use cases FE:**
- Menu systems (menu > submenu > items)
- Form builders (form > section > fields)
- File explorers (folder > subfolder > files)
- Component trees
- Comment threads (comment > replies > nested replies)

---

### 3. **Bridge Pattern** (Structural)
Tách abstraction khỏi implementation

```javascript
// Implementation hierarchy
class Renderer {
  render(data) {}
}

class CanvasRenderer extends Renderer {
  render(data) {
    console.log('Rendering to Canvas:', data);
  }
}

class SVGRenderer extends Renderer {
  render(data) {
    console.log('Rendering to SVG:', data);
  }
}

// Abstraction hierarchy
class Chart {
  constructor(renderer) {
    this.renderer = renderer;
  }
  
  draw(data) {
    this.renderer.render(data);
  }
}

class BarChart extends Chart {
  draw(data) {
    const formatted = this.formatBars(data);
    super.draw(formatted);
  }
  
  formatBars(data) {
    return data; // Format logic
  }
}

// Usage - can switch renderer independently
const chart1 = new BarChart(new CanvasRenderer());
const chart2 = new BarChart(new SVGRenderer());
```

**Use cases FE:**
- Charts (chart type vs render engine)
- Themes (component vs theme implementation)
- Storage (logic vs storage backend: localStorage/indexedDB/API)
- Notifications (notification type vs delivery method)

---

### 4. **Flyweight Pattern** (Structural)
Tiết kiệm memory bằng cách share common data

```javascript
// Flyweight factory
class IconFactory {
  constructor() {
    this.icons = {};
  }
  
  getIcon(type) {
    if (!this.icons[type]) {
      // Load icon once
      this.icons[type] = {
        type,
        svg: this.loadSVG(type) // Heavy data
      };
    }
    return this.icons[type];
  }
  
  loadSVG(type) {
    return `<svg>...${type}...</svg>`;
  }
}

// Context (unique per instance)
class FileItem {
  constructor(name, type, iconFactory) {
    this.name = name; // Unique
    this.type = type; // Unique
    this.icon = iconFactory.getIcon(type); // Shared!
  }
  
  render() {
    return `${this.icon.svg} ${this.name}`;
  }
}

// Usage
const factory = new IconFactory();

// Thousands of files, but only 3 icon objects in memory
const files = [
  new FileItem('doc1.pdf', 'pdf', factory),
  new FileItem('doc2.pdf', 'pdf', factory), // Reuses pdf icon
  new FileItem('img1.jpg', 'image', factory),
  new FileItem('img2.jpg', 'image', factory), // Reuses image icon
  // ... 10,000 files
];
```

**Use cases FE:**
- Icon libraries (share SVG data)
- Text editor (share character formatting)
- Game sprites (share texture data)
- Map markers (thousands of markers, few icon types)

---

### 5. **Chain of Responsibility** (Behavioral)
Pass request through chain until handled

```javascript
// Handler interface
class ValidationHandler {
  setNext(handler) {
    this.next = handler;
    return handler; // For chaining
  }
  
  handle(request) {
    if (this.next) {
      return this.next.handle(request);
    }
    return true;
  }
}

// Concrete handlers
class RequiredFieldValidator extends ValidationHandler {
  handle(request) {
    if (!request.value || request.value.trim() === '') {
      return { valid: false, error: 'Field is required' };
    }
    return super.handle(request);
  }
}

class EmailFormatValidator extends ValidationHandler {
  handle(request) {
    if (!/\S+@\S+\.\S+/.test(request.value)) {
      return { valid: false, error: 'Invalid email format' };
    }
    return super.handle(request);
  }
}

class EmailDomainValidator extends ValidationHandler {
  handle(request) {
    if (request.value.endsWith('@spam.com')) {
      return { valid: false, error: 'Spam domain not allowed' };
    }
    return super.handle(request);
  }
}

// Usage - build chain
const validator = new RequiredFieldValidator();
validator
  .setNext(new EmailFormatValidator())
  .setNext(new EmailDomainValidator());

// Process
console.log(validator.handle({ value: '' })); 
// { valid: false, error: 'Field is required' }

console.log(validator.handle({ value: 'invalid' }));
// { valid: false, error: 'Invalid email format' }

console.log(validator.handle({ value: 'test@spam.com' }));
// { valid: false, error: 'Spam domain not allowed' }

console.log(validator.handle({ value: 'test@gmail.com' }));
// true
```

**Use cases FE:**
- Form validation chains
- Middleware systems
- Event bubbling handlers
- Authentication/Authorization checks
- Logging/Error handling pipeline

---

### 6. **Iterator Pattern** (Behavioral)
Access elements sequentially without exposing structure

```javascript
class Collection {
  constructor(items) {
    this.items = items;
  }
  
  // Custom iterator
  [Symbol.iterator]() {
    let index = 0;
    const items = this.items;
    
    return {
      next() {
        if (index < items.length) {
          return { 
            value: items[index++], 
            done: false 
          };
        }
        return { done: true };
      }
    };
  }
  
  // Filtered iterator
  *filterIterator(predicate) {
    for (const item of this.items) {
      if (predicate(item)) {
        yield item;
      }
    }
  }
}

// Usage
const users = new Collection([
  { name: 'John', age: 25 },
  { name: 'Jane', age: 30 },
  { name: 'Bob', age: 20 }
]);

// Standard iteration
for (const user of users) {
  console.log(user.name);
}

// Filtered iteration
for (const user of users.filterIterator(u => u.age > 22)) {
  console.log(user.name); // John, Jane
}
```

**Use cases FE:**
- Paginated data traversal
- Tree traversal (DFS, BFS)
- Custom collection iteration
- Stream processing

---

### 7. **Mediator Pattern** (Behavioral)
Central communication hub, reduce coupling

```javascript
// Mediator
class ChatRoom {
  constructor() {
    this.users = new Map();
  }
  
  register(user) {
    this.users.set(user.name, user);
    user.setChatRoom(this);
  }
  
  send(message, from, to) {
    if (to) {
      // Private message
      const recipient = this.users.get(to);
      recipient?.receive(message, from);
    } else {
      // Broadcast
      this.users.forEach(user => {
        if (user.name !== from) {
          user.receive(message, from);
        }
      });
    }
  }
}

// Colleague
class User {
  constructor(name) {
    this.name = name;
    this.chatRoom = null;
  }
  
  setChatRoom(chatRoom) {
    this.chatRoom = chatRoom;
  }
  
  send(message, to = null) {
    console.log(`${this.name} sends: ${message}`);
    this.chatRoom.send(message, this.name, to);
  }
  
  receive(message, from) {
    console.log(`${this.name} receives from ${from}: ${message}`);
  }
}

// Usage
const chatRoom = new ChatRoom();

const john = new User('John');
const jane = new User('Jane');
const bob = new User('Bob');

chatRoom.register(john);
chatRoom.register(jane);
chatRoom.register(bob);

john.send('Hello everyone!'); // Broadcast
jane.send('Hi John!', 'John'); // Private
```

**Use cases FE:**
- Component communication (thay vì prop drilling)
- Event bus systems
- Form field coordination
- Modal/Dialog management
- WebSocket message routing

---

### 8. **Memento Pattern** (Behavioral)
Save and restore object state (Undo/Redo)

```javascript
// Memento
class EditorMemento {
  constructor(content, cursorPosition) {
    this.content = content;
    this.cursorPosition = cursorPosition;
  }
}

// Originator
class TextEditor {
  constructor() {
    this.content = '';
    this.cursorPosition = 0;
  }
  
  type(text) {
    this.content += text;
    this.cursorPosition += text.length;
  }
  
  save() {
    return new EditorMemento(this.content, this.cursorPosition);
  }
  
  restore(memento) {
    this.content = memento.content;
    this.cursorPosition = memento.cursorPosition;
  }
}

// Caretaker
class History {
  constructor() {
    this.mementos = [];
    this.currentIndex = -1;
  }
  
  push(memento) {
    this.mementos = this.mementos.slice(0, this.currentIndex + 1);
    this.mementos.push(memento);
    this.currentIndex++;
  }
  
  undo() {
    if (this.currentIndex > 0) {
      this.currentIndex--;
      return this.mementos[this.currentIndex];
    }
    return null;
  }
  
  redo() {
    if (this.currentIndex < this.mementos.length - 1) {
      this.currentIndex++;
      return this.mementos[this.currentIndex];
    }
    return null;
  }
}

// Usage
const editor = new TextEditor();
const history = new History();

editor.type('Hello');
history.push(editor.save());

editor.type(' World');
history.push(editor.save());

editor.type('!!!');
console.log(editor.content); // "Hello World!!!"

const previous = history.undo();
editor.restore(previous);
console.log(editor.content); // "Hello World"
```

**Use cases FE:**
- Undo/Redo functionality
- Form state snapshots
- Canvas/Drawing apps
- Game state saves
- Time travel debugging

---

### 9. **State Pattern** (Behavioral)
Object behavior changes based on internal state

```javascript
// State interface
class PlayerState {
  play(player) {}
  pause(player) {}
  stop(player) {}
}

// Concrete states
class PlayingState extends PlayerState {
  play(player) {
    console.log('Already playing');
  }
  
  pause(player) {
    console.log('Pausing...');
    player.setState(new PausedState());
  }
  
  stop(player) {
    console.log('Stopping...');
    player.setState(new StoppedState());
  }
}

class PausedState extends PlayerState {
  play(player) {
    console.log('Resuming...');
    player.setState(new PlayingState());
  }
  
  pause(player) {
    console.log('Already paused');
  }
  
  stop(player) {
    console.log('Stopping...');
    player.setState(new StoppedState());
  }
}

class StoppedState extends PlayerState {
  play(player) {
    console.log('Starting playback...');
    player.setState(new PlayingState());
  }
  
  pause(player) {
    console.log('Cannot pause when stopped');
  }
  
  stop(player) {
    console.log('Already stopped');
  }
}

// Context
class MusicPlayer {
  constructor() {
    this.state = new StoppedState();
  }
  
  setState(state) {
    this.state = state;
  }
  
  play() {
    this.state.play(this);
  }
  
  pause() {
    this.state.pause(this);
  }
  
  stop() {
    this.state.stop(this);
  }
}

// Usage
const player = new MusicPlayer();
player.play();  // Starting playback...
player.pause(); // Pausing...
player.pause(); // Already paused
player.play();  // Resuming...
player.stop();  // Stopping...
```

**Use cases FE:**
- Media players (playing/paused/stopped)
- Form states (draft/submitted/validated)
- Connection states (connecting/connected/disconnected)
- Game states (menu/playing/paused/gameover)
- Order states (pending/processing/shipped/delivered)

---

### 10. **Visitor Pattern** (Behavioral)
Add operations to object structures without modifying them

```javascript
// Element interface
class FormElement {
  accept(visitor) {
    throw new Error('Must implement accept');
  }
}

// Concrete elements
class TextInput extends FormElement {
  constructor(value) {
    super();
    this.value = value;
  }
  
  accept(visitor) {
    return visitor.visitTextInput(this);
  }
}

class Checkbox extends FormElement {
  constructor(checked) {
    super();
    this.checked = checked;
  }
  
  accept(visitor) {
    return visitor.visitCheckbox(this);
  }
}

// Visitors (different operations)
class ValidationVisitor {
  visitTextInput(input) {
    return input.value.length > 0;
  }
  
  visitCheckbox(checkbox) {
    return checkbox.checked;
  }
}

class SerializationVisitor {
  visitTextInput(input) {
    return { type: 'text', value: input.value };
  }
  
  visitCheckbox(checkbox) {
    return { type: 'checkbox', checked: checkbox.checked };
  }
}

// Usage
const form = [
  new TextInput('John Doe'),
  new Checkbox(true),
  new TextInput('')
];

const validator = new ValidationVisitor();
const serializer = new SerializationVisitor();

// Validate
form.forEach(element => {
  console.log('Valid:', element.accept(validator));
});

// Serialize
const serialized = form.map(element => 
  element.accept(serializer)
);
console.log(serialized);
```

**Use cases FE:**
- AST traversal (compilers, parsers)
- Form validation with multiple validators
- Serialization/Deserialization
- Reporting/Analytics on data structures
- Code formatters

---

## 🎯 Non-GOF Patterns quan trọng cho FE

### 11. **Module Pattern** (Classic)
```javascript
const CartModule = (function() {
  // Private
  let items = [];
  
  function calculateTotal() {
    return items.reduce((sum, item) => sum + item.price, 0);
  }
  
  // Public API
  return {
    addItem(item) {
      items.push(item);
    },
    
    getTotal() {
      return calculateTotal();
    },
    
    getItemCount() {
      return items.length;
    }
  };
})();
```

---

### 12. **Revealing Module Pattern**
```javascript
const UserModule = (function() {
  let users = [];
  
  function validateEmail(email) {
    return /\S+@\S+/.test(email);
  }
  
  function addUser(user) {
    if (validateEmail(user.email)) {
      users.push(user);
      return true;
    }
    return false;
  }
  
  function getUsers() {
    return [...users]; // Return copy
  }
  
  // Reveal only what's needed
  return {
    add: addUser,
    getAll: getUsers
  };
})();
```

---

### 13. **Mixin Pattern**
```javascript
// Mixins
const timestampMixin = {
  setCreatedAt() {
    this.createdAt = new Date();
  },
  
  setUpdatedAt() {
    this.updatedAt = new Date();
  }
};

const validationMixin = {
  validate() {
    return this.name && this.email;
  }
};

// Apply mixins
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

Object.assign(User.prototype, timestampMixin, validationMixin);

// Usage
const user = new User('John', 'john@example.com');
user.setCreatedAt();
console.log(user.validate()); // true
```

---

### 14. **Dependency Injection Pattern**
```javascript
// Bad: Hard-coded dependencies
class UserService {
  constructor() {
    this.api = new APIClient(); // Tightly coupled
    this.logger = new Logger();
  }
}

// Good: Injected dependencies
class UserService {
  constructor(api, logger) {
    this.api = api;
    this.logger = logger;
  }
  
  async getUser(id) {
    this.logger.log(`Fetching user ${id}`);
    return await this.api.get(`/users/${id}`);
  }
}

// Usage - easy to test & swap implementations
const service = new UserService(
  new APIClient(),
  new ConsoleLogger()
);

// In tests
const testService = new UserService(
  new MockAPI(),
  new MockLogger()
);
```

---

### 15. **Registry Pattern**
```javascript
class ComponentRegistry {
  constructor() {
    this.components = new Map();
  }
  
  register(name, component) {
    this.components.set(name, component);
  }
  
  get(name) {
    if (!this.components.has(name)) {
      throw new Error(`Component ${name} not registered`);
    }
    return this.components.get(name);
  }
  
  has(name) {
    return this.components.has(name);
  }
}

// Global registry
const registry = new ComponentRegistry();

registry.register('Button', ButtonComponent);
registry.register('Input', InputComponent);

// Usage
const Button = registry.get('Button');
```

**Use cases:**
- Component libraries
- Plugin systems
- Route registration
- Event type registration

---

### 16. **Lazy Initialization Pattern**
```javascript
class ExpensiveResource {
  constructor() {
    this._instance = null;
  }
  
  get instance() {
    if (!this._instance) {
      console.log('Creating expensive resource...');
      this._instance = this.createResource();
    }
    return this._instance;
  }
  
  createResource() {
    // Heavy initialization
    return { data: 'expensive data' };
  }
}

// Only created when first accessed
const resource = new ExpensiveResource();
console.log('Resource object created');
// ... much later
console.log(resource.instance); // Now initialized
```

---

### 17. **Object Pool Pattern**
```javascript
class ConnectionPool {
  constructor(size) {
    this.size = size;
    this.available = [];
    this.inUse = [];
    
    // Pre-create connections
    for (let i = 0; i < size; i++) {
      this.available.push(this.createConnection());
    }
  }
  
  createConnection() {
    return { id: Math.random(), query: () => {} };
  }
  
  acquire() {
    if (this.available.length === 0) {
      throw new Error('No connections available');
    }
    
    const conn = this.available.pop();
    this.inUse.push(conn);
    return conn;
  }
  
  release(conn) {
    const index = this.inUse.indexOf(conn);
    if (index > -1) {
      this.inUse.splice(index, 1);
      this.available.push(conn);
    }
  }
}

// Usage
const pool = new ConnectionPool(5);

const conn1 = pool.acquire();
conn1.query();
pool.release(conn1); // Reuse
```

**Use cases:**
- WebSocket connections
- Canvas contexts
- Heavy DOM elements
- Worker threads

---

## 📋 Tổng kết đề xuất bổ sung

### Ưu tiên cao (⭐⭐⭐):
1. **Composite** - Cực kỳ quan trọng cho UI trees
2. **Chain of Responsibility** - Validation, middleware
3. **State Pattern** - State machines everywhere
4. **Memento** - Undo/redo là must-have
5. **Dependency Injection** - Testability & flexibility

### Ưu tiên trung bình (⭐⭐):
6. **Bridge** - Separation of concerns
7. **Flyweight** - Performance optimization
8. **Mediator** - Decoupling components
9. **Prototype** - Object cloning
10. **Module/Revealing Module** - Classic JS patterns

### Ưu tiên thấp (⭐):
11. **Visitor** - Specific use cases
12. **Iterator** - JS has built-in support
13. **Registry** - Plugin systems
14. **Object Pool** - Performance edge cases

---

## 💡 Gợi ý cấu trúc lại file

```markdown
# Design Patterns for Frontend

## 1. Creational Patterns
- Singleton ✅
- Factory ✅  
- Builder ✅
- **Prototype** ← Thêm

## 2. Structural Patterns
- Adapter ✅
- Decorator ✅
- Proxy ✅
- Facade ✅
- **Composite** ← Thêm
- **Bridge** ← Thêm
- **Flyweight** ← Thêm

## 3. Behavioral Patterns
- Observer ✅
- Strategy ✅
- Command ✅
- Template Method ✅
- **Chain of Responsibility** ← Thêm
- **State** ← Thêm
- **Memento** ← Thêm
- **Mediator** ← Thêm
- **Iterator** ← Thêm
- **Visitor** ← Thêm (optional)

## 4. JavaScript-Specific Patterns
- **Module Pattern** ← Thêm
- **Revealing Module** ← Thêm
- **Mixin** ← Thêm
- **Dependency Injection** ← Thêm
- **Registry** ← Thêm
- **Lazy Initialization** ← Thêm
- **Object Pool** ← Thêm (optional)

## 5. When to Use / Anti-Patterns
← Section mới
```
