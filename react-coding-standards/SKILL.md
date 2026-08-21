---
name: react-coding-standards
description: Enforce React coding standards and best practices. Use when writing React components, managing state, handling effects, or reviewing React code to ensure compliance with modern React patterns and conventions.
allowed-tools: Read, Edit, Grep, Glob
---

# React Coding Standards Skill

A comprehensive guide for writing high-quality React applications following official React documentation best practices.

## Core Principles

React is built on three fundamental idioms:

1. **Components and Hooks Must Be Pure** - Easier to understand, debug, and enables automatic optimizations
2. **React Controls Execution** - Let React determine when to render and optimize
3. **Rules of Hooks** - Follow specific calling patterns enforced by ESLint

---

## 1. Component Fundamentals

### Component Definition

```jsx
// ✅ Correct: Function components with capital letter
function MyButton() {
  return <button>Click me</button>;
}

// ✅ Correct: Arrow function components
const MyButton = () => {
  return <button>Click me</button>;
};

// ❌ Wrong: Lowercase component names
function myButton() {
  return <button>Click me</button>;
}
```

### JSX Rules

```jsx
// ✅ Correct: Self-closing tags
<img src={url} alt="description" />
<br />
<input type="text" />

// ✅ Correct: Multiple elements wrapped in fragment
return (
  <>
    <Header />
    <Main />
    <Footer />
  </>
);

// ✅ Correct: Use className for CSS
<div className="container">

// ❌ Wrong: Using class (HTML attribute)
<div class="container">

// ✅ Correct: Inline styles as objects
<div style={{ width: '100px', backgroundColor: 'blue' }}>

// ❌ Wrong: Inline styles as strings
<div style="width: 100px">
```

### Displaying Data

```jsx
// ✅ Correct: Embedding variables with curly braces
function UserProfile({ user }) {
  return (
    <div>
      <h1>{user.name}</h1>
      <img src={user.avatarUrl} alt={user.name} />
    </div>
  );
}
```

---

## 2. Conditional Rendering

```jsx
// ✅ Correct: Ternary operator
function Greeting({ isLoggedIn }) {
  return isLoggedIn ? <AdminPanel /> : <LoginForm />;
}

// ✅ Correct: Logical AND for conditional display
function Notification({ hasMessages }) {
  return (
    <div>
      {hasMessages && <Badge count={messageCount} />}
    </div>
  );
}

// ✅ Correct: Early return pattern
function Profile({ user }) {
  if (!user) {
    return <LoadingSpinner />;
  }
  return <UserDetails user={user} />;
}

// ❌ Wrong: Using && with numbers (0 will render)
{count && <Message />}  // Renders "0" when count is 0

// ✅ Correct: Explicit boolean conversion
{count > 0 && <Message />}
```

---

## 3. Rendering Lists

```jsx
// ✅ Correct: Always provide unique keys
function ProductList({ products }) {
  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          {product.name} - ${product.price}
        </li>
      ))}
    </ul>
  );
}

// ❌ Wrong: Using array index as key (unstable)
{items.map((item, index) => (
  <Item key={index} data={item} />  // Don't do this if list can reorder
))}

// ❌ Wrong: Missing keys
{items.map((item) => (
  <Item data={item} />  // React will warn about missing key
))}

// ✅ Correct: Generate stable keys if none exist
{items.map((item) => (
  <Item key={item.name + item.category} data={item} />
))}
```

---

## 4. Event Handling

```jsx
// ✅ Correct: Pass function reference (not call result)
function Button() {
  const handleClick = () => {
    console.log('Clicked!');
  };

  return <button onClick={handleClick}>Click me</button>;
}

// ❌ Wrong: Calling the function immediately
<button onClick={handleClick()}>  // This runs on render!

// ✅ Correct: Passing arguments with arrow function
<button onClick={() => handleDelete(item.id)}>Delete</button>

// ✅ Correct: Event handler receives event object
function Input() {
  const handleChange = (e) => {
    console.log(e.target.value);
  };

  return <input onChange={handleChange} />;
}
```

---

## 5. State Management with useState

```jsx
import { useState } from 'react';

// ✅ Correct: Basic state usage
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// ✅ Correct: Functional updates for state based on previous value
function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(prevCount => prevCount + 1);
  };

  return <button onClick={increment}>Count: {count}</button>;
}

// ✅ Correct: Object state with spread operator
function Form() {
  const [form, setForm] = useState({ name: '', email: '' });

  const handleChange = (e) => {
    setForm(prev => ({
      ...prev,
      [e.target.name]: e.target.value
    }));
  };

  return (
    <>
      <input name="name" value={form.name} onChange={handleChange} />
      <input name="email" value={form.email} onChange={handleChange} />
    </>
  );
}

// ❌ Wrong: Mutating state directly
const [items, setItems] = useState([]);
items.push(newItem);  // Don't mutate!

// ✅ Correct: Create new array
setItems([...items, newItem]);
```

---

## 6. State Management with useReducer

```jsx
import { useReducer } from 'react';

// ✅ Correct: Reducer for complex state logic
function tasksReducer(state, action) {
  switch (action.type) {
    case 'added':
      return [...state, { id: action.id, text: action.text, done: false }];
    case 'changed':
      return state.map(t => t.id === action.task.id ? action.task : t);
    case 'deleted':
      return state.filter(t => t.id !== action.id);
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function TaskList() {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

  const handleAdd = (text) => {
    dispatch({ type: 'added', id: nextId++, text });
  };

  const handleDelete = (id) => {
    dispatch({ type: 'deleted', id });
  };

  return (/* ... */);
}
```

---

## 7. Context API

```jsx
import { createContext, useContext, useState } from 'react';

// ✅ Correct: Create and provide context
const ThemeContext = createContext('light');

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// ✅ Correct: Consume context with useContext
function ThemedButton() {
  const { theme, setTheme } = useContext(ThemeContext);

  return (
    <button
      className={theme}
      onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}
    >
      Toggle Theme
    </button>
  );
}

// ✅ Correct: Custom hook for context
function useTheme() {
  const context = useContext(ThemeContext);
  if (context === undefined) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return context;
}
```

---

## 8. Rules of Hooks

### Must Follow

```jsx
// ✅ Correct: Hooks at top level of component
function Component() {
  const [state, setState] = useState(0);
  const value = useContext(MyContext);

  useEffect(() => {
    // effect
  }, []);

  return <div>{state}</div>;
}

// ❌ Wrong: Hooks inside conditions
function Component({ condition }) {
  if (condition) {
    const [state, setState] = useState(0);  // Don't do this!
  }
}

// ❌ Wrong: Hooks inside loops
function Component({ items }) {
  for (const item of items) {
    const [state, setState] = useState(item);  // Don't do this!
  }
}

// ❌ Wrong: Hooks inside nested functions
function Component() {
  function handleClick() {
    const [state, setState] = useState(0);  // Don't do this!
  }
}

// ❌ Wrong: Hooks in regular JavaScript functions
function regularFunction() {
  const [state, setState] = useState(0);  // Don't do this!
}
```

---

## 9. Effects with useEffect

### Basic Usage

```jsx
import { useEffect } from 'react';

// ✅ Correct: Effect with cleanup
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();

    return () => {
      connection.disconnect();  // Cleanup function
    };
  }, [roomId]);  // Re-run when roomId changes

  return <div>Chat Room: {roomId}</div>;
}

// ✅ Correct: Effect that runs once on mount
useEffect(() => {
  fetchInitialData();
}, []);  // Empty dependency array

// ✅ Correct: Effect that runs on every render
useEffect(() => {
  document.title = `Count: ${count}`;
});  // No dependency array
```

### Anti-Patterns to Avoid

```jsx
// ❌ Wrong: Using effect for data transformation
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [fullName, setFullName] = useState('');

useEffect(() => {
  setFullName(firstName + ' ' + lastName);  // Unnecessary effect
}, [firstName, lastName]);

// ✅ Correct: Calculate during render
const fullName = firstName + ' ' + lastName;

// ❌ Wrong: Effect for user event handling
useEffect(() => {
  if (submitted) {
    sendData(formData);
  }
}, [submitted, formData]);

// ✅ Correct: Handle in event handler
const handleSubmit = () => {
  sendData(formData);
};
```

---

## 10. Refs with useRef

```jsx
import { useRef } from 'react';

// ✅ Correct: DOM element reference
function TextInput() {
  const inputRef = useRef(null);

  const focusInput = () => {
    inputRef.current.focus();
  };

  return (
    <>
      <input ref={inputRef} />
      <button onClick={focusInput}>Focus Input</button>
    </>
  );
}

// ✅ Correct: Storing mutable values that don't trigger re-render
function Timer() {
  const intervalRef = useRef(null);

  const startTimer = () => {
    intervalRef.current = setInterval(() => {
      // ...
    }, 1000);
  };

  const stopTimer = () => {
    clearInterval(intervalRef.current);
  };

  return (/* ... */);
}

// ❌ Wrong: Using ref for values that should trigger re-render
const countRef = useRef(0);
countRef.current++;  // UI won't update!

// ✅ Correct: Use state for values that affect UI
const [count, setCount] = useState(0);
setCount(c => c + 1);  // UI updates
```

---

## 11. Performance Optimization

### useMemo

```jsx
import { useMemo } from 'react';

// ✅ Correct: Cache expensive calculations
function FilteredList({ items, filter }) {
  const filteredItems = useMemo(() => {
    return items.filter(item => item.name.includes(filter));
  }, [items, filter]);

  return (
    <ul>
      {filteredItems.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}

// ❌ Wrong: Memoizing cheap operations
const doubled = useMemo(() => count * 2, [count]);  // Unnecessary

// ✅ Correct: Just calculate directly
const doubled = count * 2;
```

### useCallback

```jsx
import { useCallback } from 'react';

// ✅ Correct: Cache function for child component optimization
function Parent({ items }) {
  const handleClick = useCallback((id) => {
    console.log('Clicked:', id);
  }, []);

  return (
    <List items={items} onItemClick={handleClick} />
  );
}

// Only useful when passing to memoized child
const MemoizedList = memo(function List({ items, onItemClick }) {
  return items.map(item => (
    <button key={item.id} onClick={() => onItemClick(item.id)}>
      {item.name}
    </button>
  ));
});
```

### React.memo

```jsx
import { memo } from 'react';

// ✅ Correct: Memoize component that receives same props often
const ExpensiveComponent = memo(function ExpensiveComponent({ data }) {
  // Expensive rendering logic
  return <div>{/* ... */}</div>;
});

// ❌ Wrong: Memoizing components that always receive new props
const ListItem = memo(function ListItem({ item, onClick }) {
  return <div onClick={() => onClick(item.id)}>{item.name}</div>;
});
// onClick is recreated every render, defeating memo
```

---

## 12. Custom Hooks

```jsx
// ✅ Correct: Custom hook for reusable logic
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue];
}

// Usage
function App() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  return <div className={theme}>...</div>;
}

// ✅ Correct: Custom hook for window size
function useWindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };

    handleResize();
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}
```

---

## 13. Props Best Practices

```jsx
// ✅ Correct: Destructure props
function UserCard({ name, email, avatar }) {
  return (
    <div>
      <img src={avatar} alt={name} />
      <h2>{name}</h2>
      <p>{email}</p>
    </div>
  );
}

// ✅ Correct: Default props with destructuring
function Button({ variant = 'primary', size = 'medium', children }) {
  return (
    <button className={`btn btn-${variant} btn-${size}`}>
      {children}
    </button>
  );
}

// ✅ Correct: Spread remaining props
function Input({ label, ...props }) {
  return (
    <label>
      {label}
      <input {...props} />
    </label>
  );
}

// ✅ Correct: Children prop for composition
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
}
```

---

## 14. Component Composition

### Lifting State Up

```jsx
// ✅ Correct: Shared state in parent
function FilterableList() {
  const [filter, setFilter] = useState('');
  const [items] = useState(initialItems);

  const filteredItems = items.filter(item =>
    item.name.includes(filter)
  );

  return (
    <>
      <SearchBar filter={filter} onFilterChange={setFilter} />
      <ItemList items={filteredItems} />
    </>
  );
}

function SearchBar({ filter, onFilterChange }) {
  return (
    <input
      value={filter}
      onChange={(e) => onFilterChange(e.target.value)}
      placeholder="Search..."
    />
  );
}

function ItemList({ items }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}
```

### Render Props Pattern

```jsx
// ✅ Correct: Render props for flexible rendering
function MouseTracker({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);

  return render(position);
}

// Usage
<MouseTracker render={({ x, y }) => (
  <p>Mouse position: {x}, {y}</p>
)} />
```

---

## 15. State Design Principles

### Minimal State

```jsx
// ❌ Wrong: Redundant state
function ProductList({ products }) {
  const [items, setItems] = useState(products);
  const [filteredItems, setFilteredItems] = useState(products);
  const [itemCount, setItemCount] = useState(products.length);

  // All three are redundant!
}

// ✅ Correct: Derive from minimal state
function ProductList({ products }) {
  const [filter, setFilter] = useState('');

  // Derived values - not state
  const filteredItems = products.filter(p => p.name.includes(filter));
  const itemCount = filteredItems.length;

  return (/* ... */);
}
```

### Avoid Redundant State

```jsx
// ❌ Wrong: Storing computed values
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [fullName, setFullName] = useState('');  // Redundant!

// ✅ Correct: Compute during render
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const fullName = `${firstName} ${lastName}`;  // Computed
```

### Avoid Duplication

```jsx
// ❌ Wrong: Duplicated item data
const [items, setItems] = useState(initialItems);
const [selectedItem, setSelectedItem] = useState(items[0]);  // Duplication!

// ✅ Correct: Store ID only
const [items, setItems] = useState(initialItems);
const [selectedId, setSelectedId] = useState(items[0]?.id);
const selectedItem = items.find(item => item.id === selectedId);
```

---

## 16. Immutability Patterns

```jsx
// ✅ Correct: Updating arrays immutably
// Adding
setItems([...items, newItem]);

// Removing
setItems(items.filter(item => item.id !== idToRemove));

// Updating
setItems(items.map(item =>
  item.id === idToUpdate ? { ...item, ...updates } : item
));

// ✅ Correct: Updating nested objects
setUser(prev => ({
  ...prev,
  address: {
    ...prev.address,
    city: newCity
  }
}));

// ❌ Wrong: Direct mutation
items.push(newItem);
items[0].name = 'New Name';
user.address.city = 'New City';
```

---

## Code Checklist

### Before Writing Code

- [ ] Understand the component's single responsibility
- [ ] Identify what state is truly needed (minimal state)
- [ ] Determine where state should live (closest common ancestor)
- [ ] Plan component composition and data flow

### While Writing Code

- [ ] Hooks at top level of components only
- [ ] All dependencies included in useEffect arrays
- [ ] Cleanup functions for effects with subscriptions
- [ ] Keys on all list items
- [ ] No direct state mutation
- [ ] Event handlers pass functions, not calls

### After Writing Code

- [ ] No unnecessary effects (can it be computed during render?)
- [ ] No redundant state
- [ ] Performance hooks (useMemo, useCallback) used appropriately
- [ ] Custom hooks extracted for reusable logic
- [ ] ESLint rules of hooks passing

---

## Common Anti-Patterns

### 1. Effect for Event Handling

```jsx
// ❌ Wrong
const [submitted, setSubmitted] = useState(false);
useEffect(() => {
  if (submitted) {
    submitForm(data);
  }
}, [submitted, data]);

// ✅ Correct
const handleSubmit = () => {
  submitForm(data);
};
```

### 2. Prop Drilling

```jsx
// ❌ Wrong: Passing through many levels
<App user={user}>
  <Header user={user}>
    <Navigation user={user}>
      <UserMenu user={user} />  // Finally used here
    </Navigation>
  </Header>
</App>

// ✅ Correct: Use context
const UserContext = createContext(null);
<UserContext.Provider value={user}>
  <App />
</UserContext.Provider>
```

### 3. Calling Components as Functions

```jsx
// ❌ Wrong: Direct function call
function Parent() {
  return <div>{Child()}</div>;
}

// ✅ Correct: JSX syntax
function Parent() {
  return <div><Child /></div>;
}
```

---

## References

- [React Official Documentation](https://react.dev/)
- [React Learn Section](https://react.dev/learn)
- [React Hooks Reference](https://react.dev/reference/react/hooks)
- [Rules of React](https://react.dev/reference/rules)

---

## Using This Skill

Use this skill when you need to:

- Write new React components
- Manage state effectively
- Handle side effects properly
- Optimize component performance
- Review React code for best practices
- Refactor existing React code

Claude will automatically apply these standards to ensure generated React code follows modern best practices.
