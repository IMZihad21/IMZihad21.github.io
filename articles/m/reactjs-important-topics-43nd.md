# Getting Started with React.js: A Beginner's Guide

- Canonical URL: https://imzihad21.github.io/articles/a/reactjs-important-topics-43nd/
- Source URL: https://dev.to/imzihad21/reactjs-important-topics-43nd
- Web View: https://imzihad21.github.io/articles/a/reactjs-important-topics-43nd/
- Published: 2021-12-22T17:43:19.000Z
- Modified: 2021-12-22T17:43:19.000Z
- Reading time: 3 minutes
- Tags: javascript, webdev, beginners, react

React.js is a popular JavaScript library for building user interfaces, especially single-page applications. This guide is aligned with modern React 19 development and current official feature updates.

As of March 17, 2026, React 19.2 (October 1, 2025) is the latest major feature release. If you use React Server Components packages, apply the December 2025 security patch line (19.2.1 or newer).

### Why It Matters

- React helps you build reusable UI blocks instead of repeating markup everywhere.
- It keeps UI updates predictable with one-way data flow.
- React 19 improves forms, async UX, and rendering behavior.
- The ecosystem is mature, and modern tooling support is excellent.

### Core Concepts

#### 1. Components

Components are the core building blocks of a React app. Functional components are the modern default. Class components mostly appear in legacy codebases.

```javascript
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

```javascript
class LegacyGreeting extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

#### 2. JSX

JSX lets you write HTML-like syntax inside JavaScript, so UI structure is easier to read and maintain.

```javascript
const headingElement = <h1>This is JSX</h1>;
```

JSX is transpiled into plain JavaScript by tools like Babel.

#### 3. Props

Props pass data from parent components to child components. Treat props as read-only inputs.

```javascript
function App() {
  return <Greeting name="Alice" />;
}
```

#### 4. State

State stores local data that can change over time. When state changes, React updates the UI.

```javascript
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount((previousCount) => previousCount + 1)}>
        Increment
      </button>
    </div>
  );
}
```

#### 5. Effects and Events

Use `useEffect` for synchronization with external systems. In React 19.2, `useEffectEvent` helps keep event logic fresh without forcing effect re-subscriptions.

```javascript
import { useEffect, useEffectEvent, useState } from "react";

function NetworkStatus() {
  const [status, setStatus] = useState(navigator.onLine ? "online" : "offline");

  const onStatusChange = useEffectEvent(() => {
    setStatus(navigator.onLine ? "online" : "offline");
  });

  useEffect(() => {
    window.addEventListener("online", onStatusChange);
    window.addEventListener("offline", onStatusChange);

    return () => {
      window.removeEventListener("online", onStatusChange);
      window.removeEventListener("offline", onStatusChange);
    };
  }, []);

  return <p>Network: {status}</p>;
}
```

#### 6. Modern React 19 Features

Important React 19 and 19.2 features to know:

- Actions and `useActionState` for async form workflows.
- `useOptimistic` for instant UI updates while async work is in progress.
- `<Activity />` for controlling and prioritizing app sections.
- `cacheSignal` and React Performance Tracks for better rendering control and profiling.
- Partial pre-rendering and resume APIs for modern server rendering workflows.

#### 7. Context API

Context API helps share values like theme or authenticated user across many components without prop drilling.

```javascript
import { createContext, useContext } from "react";

const AppContext = createContext({ currentUser: null });

function UserBadge() {
  const { currentUser } = useContext(AppContext);
  return <span>{currentUser ?? "Guest"}</span>;
}
```

#### 8. Virtual DOM

The Virtual DOM is an in-memory representation of the real DOM. React compares changes and updates only what is needed.

### Practical Example

Start a React app with Vite:

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

Use modern root rendering in `main.jsx`:

```javascript
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

If it runs on first try, that is not luck. That is engineering and maybe one tiny miracle.

### Common Mistakes

- Mutating state directly instead of using state setters.
- Adding wrong dependencies to effects and creating re-run loops.
- Forcing class lifecycle patterns into hook-first code.
- Using global context for everything instead of keeping state local when possible.
- Ignoring React security advisories when using Server Components packages.

### Quick Recap

- Functional components are the modern React default.
- Props are read-only, state changes via setter functions.
- Effects sync external systems; `useEffectEvent` helps event logic stay clean.
- React 19 improves forms, optimistic UX, and render management.
- Server and performance features in 19.2 support scalable production apps.

### Next Steps

1. Build a small todo app with add, remove, edit, and filter features.
2. Add API integration with loading, error, and optimistic UI states.
3. Try `useActionState` or `useOptimistic` in one real form flow.
4. Track release notes and security updates on the official React blog.