# React Class Notes: Routing, API Integration, and UI & Performance

*Follows State Management (Context, Redux, Redux Toolkit). Suggested class length: about 2 hours including exercises.*

## Table of Contents

- [Class Agenda](#class-agenda)
- [Part 1: Routing](#part-1-routing)
- [Part 2: API Integration](#part-2-api-integration)
- [Part 3: UI & Performance Handling](#part-3-ui--performance-handling)
- [Hands-On Exercise](#hands-on-exercise-20-to-25-minutes)
- [Quick Recap](#quick-recap)
- [Check-for-Understanding Questions](#check-for-understanding-questions)
- [Reference Links](#reference-links)

---

## Class Agenda

1. Routing with React Router (about 40 min)
2. API Integration: fetching data, loading and error states (about 35 min)
3. UI & Performance: lazy loading, code splitting, error boundaries, CSR vs SSR (about 35 min)
4. Hands-on exercise and recap (about 10 min)

**By the end of class, students should be able to:**

- Build a multi-page single page application (SPA) with nested and dynamic routes.
- Fetch data from an API and handle loading and error states properly.
- Split a bundle, lazy load routes, and catch rendering errors with an error boundary.
- Explain the difference between client-side and server-side rendering.

---

## Part 1: Routing

### 1.1 Why routing?

A React app is a **single page application**: the browser loads one HTML file and JavaScript swaps what is shown on screen. Without a router, the URL never changes, so there are no bookmarks, no back button, and no shareable links.

**Client-side routing** fixes this. The router watches the URL, decides which component to render, and updates the address bar *without a full page reload*.

### 1.2 Setup

```bash
npm install react-router-dom
```

> [!NOTE]
> These notes use React Router v6 and later. In v7, the same components can be imported from `react-router`, but `react-router-dom` still works. Older tutorials use `Switch` and `component={...}`, which were replaced by `Routes` and `element={...}`.

### 1.3 Types of routers

| Router | URL looks like | Use when |
|---|---|---|
| `BrowserRouter` | `/about` | Default choice for web apps. Uses the History API. Server must send `index.html` for every path. |
| `HashRouter` | `/#/about` | Static hosting with no server configuration (e.g. simple GitHub Pages setups). |
| `MemoryRouter` | none (kept in memory) | Tests, React Native, embedded widgets. |
| `StaticRouter` | set by server | Server-side rendering. |
| `createBrowserRouter` + `RouterProvider` | `/about` | The modern "data router" API that supports loaders, actions, and error elements. |

### 1.4 The core pieces

```jsx
import { BrowserRouter, Routes, Route, Link } from "react-router-dom";

function Home()  { return <h1>Home</h1>; }
function About() { return <h1>About</h1>; }
function NotFound() { return <h1>404 - Page not found</h1>; }

export default function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link> | <Link to="/about">About</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}
```

**What each part does:**

- `BrowserRouter` wraps the app and provides routing context (it is Context API under the hood, which connects nicely to last class).
- `Routes` looks at the current URL and renders the **best matching** `Route`.
- `Route` maps a `path` to an `element`.
- `path="*"` matches anything left over, which makes it the 404 page.
- `Link` navigates without reloading the page.

**Teaching point:** Replace `<Link>` with a plain `<a href>` in the demo and open the Network tab. The `<a>` triggers a full page reload, and `<Link>` does not.

### 1.5 `Link` vs `NavLink`

`NavLink` is a `Link` that knows whether it is the active page. It is the right choice for menus.

```jsx
import { NavLink } from "react-router-dom";

<NavLink
  to="/about"
  className={({ isActive }) => (isActive ? "nav-link active" : "nav-link")}
>
  About
</NavLink>
```

By default, `NavLink` also adds an `active` CSS class to the matching link. Use the `end` prop on `to="/"` so Home is not highlighted on every page.

### 1.6 Dynamic routes and URL parameters

```jsx
<Route path="/users/:userId" element={<UserProfile />} />
```

```jsx
import { useParams } from "react-router-dom";

function UserProfile() {
  const { userId } = useParams();   // always a string
  return <h2>Profile of user {userId}</h2>;
}
```

Visiting `/users/42` renders `Profile of user 42`. Remember: params are **strings**, so convert with `Number(userId)` when needed.

### 1.7 Nested routes and `Outlet`

A shared layout (header, sidebar) can stay on screen while only the inner part changes.

```jsx
import { Outlet } from "react-router-dom";

function Layout() {
  return (
    <>
      <header>My App</header>
      <main>
        <Outlet />   {/* child route renders here */}
      </main>
    </>
  );
}

<Routes>
  <Route element={<Layout />}>
    <Route index element={<Home />} />
    <Route path="about" element={<About />} />
    <Route path="users/:userId" element={<UserProfile />} />
  </Route>
  <Route path="*" element={<NotFound />} />
</Routes>
```

- `index` marks the default child, shown when the parent path matches exactly.
- Child paths are **relative** to the parent (no leading slash).

### 1.8 React Router hooks

| Hook | What it gives you | Typical use |
|---|---|---|
| `useNavigate()` | A function to change the URL in code | Redirect after login or form submit |
| `useParams()` | Dynamic segments such as `:userId` | Detail pages |
| `useLocation()` | Current location (`pathname`, `search`, `state`) | Analytics, reading passed state |
| `useSearchParams()` | Read and write the query string (`?sort=asc`) | Filters, pagination, search |
| `useMatch()` | Whether a pattern matches the current URL | Custom active styling |

```jsx
import { useNavigate, useSearchParams } from "react-router-dom";

function LoginForm() {
  const navigate = useNavigate();

  function handleSubmit(e) {
    e.preventDefault();
    // ...log the user in...
    navigate("/dashboard", { replace: true }); // replace: back button skips the login page
  }
  return <form onSubmit={handleSubmit}>{/* fields */}</form>;
}

function ProductList() {
  const [params, setParams] = useSearchParams();
  const sort = params.get("sort") ?? "asc";
  return (
    <button onClick={() => setParams({ sort: sort === "asc" ? "desc" : "asc" })}>
      Sort: {sort}
    </button>
  );
}
```

### 1.9 Redirects and protected routes

```jsx
import { Navigate } from "react-router-dom";

function ProtectedRoute({ isLoggedIn, children }) {
  return isLoggedIn ? children : <Navigate to="/login" replace />;
}

<Route
  path="/dashboard"
  element={
    <ProtectedRoute isLoggedIn={user !== null}>
      <Dashboard />
    </ProtectedRoute>
  }
/>
```

**Connection to last class:** `isLoggedIn` usually comes from Context or Redux.

### 1.10 Common mistakes

- Using `<a href>` instead of `<Link>` (causes full reloads and loses app state).
- Forgetting the `*` route, so unknown URLs show a blank page.
- Using an absolute path (`/about`) inside nested routes when a relative one (`about`) was intended.
- Deploying a `BrowserRouter` app to a host that returns a 404 for deep links. The server must be configured to serve `index.html` for all paths.

---

## Part 2: API Integration

### 2.1 The big picture

Most real apps get their data from a server. In React the flow is always:

1. Component renders with **no data yet** (loading state).
2. A side effect (`useEffect`) sends the request.
3. The response arrives, and we store it in state, which triggers a re-render.
4. If something goes wrong, we store an **error** in state and show a message.

Every data fetch therefore has **three states**: *loading*, *success*, and *error*. Beginners often only handle success. Making students handle all three is the main lesson of this section.

### 2.2 HTTP requests refresher

| Method | Purpose | Example | Has body? |
|---|---|---|---|
| `GET` | Read data | Get a list of users | No |
| `POST` | Create data | Add a new user | Yes |
| `PUT` | Replace data | Replace a user record | Yes |
| `PATCH` | Partially update data | Change only a user's email | Yes |
| `DELETE` | Remove data | Delete a user | Usually no |

**Common status codes:** `200` OK, `201` Created, `400` Bad request, `401` Unauthorized, `403` Forbidden, `404` Not found, `500` Server error.

### 2.3 Fetching with the Fetch API

```jsx
import { useState, useEffect } from "react";

function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    async function loadUsers() {
      try {
        const res = await fetch("https://jsonplaceholder.typicode.com/users", {
          signal: controller.signal,
        });
        if (!res.ok) throw new Error(`Request failed: ${res.status}`);
        const data = await res.json();
        setUsers(data);
      } catch (err) {
        if (err.name !== "AbortError") setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    loadUsers();
    return () => controller.abort();   // cleanup if the component unmounts
  }, []);

  if (loading) return <p>Loading...</p>;
  if (error)   return <p style={{ color: "red" }}>Something went wrong: {error}</p>;

  return (
    <ul>
      {users.map((u) => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

**Points to stress:**

- **`fetch` does not throw on 404 or 500.** It only rejects on network failure. That is why we check `res.ok` ourselves. This is the most common bug students hit.
- The `[]` dependency array means "run once after the first render".
- `AbortController` cancels the request when the component unmounts, which avoids updating state on a component that no longer exists.
- `finally` ensures the loading flag is cleared whether the request succeeded or failed.

### 2.4 Sending data (POST)

```jsx
async function addUser(newUser) {
  const res = await fetch("https://jsonplaceholder.typicode.com/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(newUser),
  });
  if (!res.ok) throw new Error("Could not create user");
  return res.json();
}
```

### 2.5 Reusing the logic: a custom hook

The loading/error/data pattern repeats everywhere, so extract it:

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`Request failed: ${res.status}`);
        return res.json();
      })
      .then(setData)
      .catch((err) => { if (err.name !== "AbortError") setError(err.message); })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// Usage
const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
```

Because `url` is in the dependency array, navigating from `/users/1` to `/users/2` automatically refetches. This ties routing and API calls together.

### 2.6 Error handling strategies

| Kind of problem | How it shows up | What to do |
|---|---|---|
| Network failure (offline, DNS) | `fetch` rejects | `catch`, show "check your connection" with a Retry button |
| HTTP error (404, 500) | `res.ok` is `false` | Throw manually, show a friendly message |
| Bad data shape | Code crashes later while rendering | Validate or use optional chaining (`data?.name`) |
| Unexpected rendering crash | Whole UI goes blank | Error boundary (Part 3) |

**Good UX checklist:** a spinner or skeleton while loading, a clear error message with a way to retry, an empty-state message when the list is empty, and disabled submit buttons while a request is in flight.

### 2.7 Beyond `useEffect` (mention only)

Hand-written fetching gets tedious once you need caching, retries, refetch on focus, and deduplication. Libraries such as **TanStack Query** and **SWR** handle these. Redux Toolkit's **RTK Query** does the same inside Redux. Students should understand the manual approach first, then adopt a library.

---

## Part 3: UI & Performance Handling

### 3.1 Why performance matters

By default, bundlers such as Vite or webpack combine your whole app into one large JavaScript file. Users download all of it, even for pages they never visit. Large bundles mean slow first loads, especially on mobile networks.

### 3.2 Code splitting

**Code splitting** means breaking the bundle into smaller **chunks** that load only when needed. Bundlers do the splitting automatically whenever they see a dynamic `import()`.

```js
// Static import: included in the main bundle
import Chart from "./Chart";

// Dynamic import: becomes its own chunk, downloaded on demand
const module = await import("./Chart");
```

### 3.3 Lazy loading with `React.lazy` and `Suspense`

`React.lazy` wraps a dynamic import so a component loads only when first rendered. `Suspense` shows a fallback while it downloads.

```jsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

const Home      = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings  = lazy(() => import("./pages/Settings"));

export default function App() {
  return (
    <Suspense fallback={<p>Loading page...</p>}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

**Rules to remember:**

- The lazy component must be a **default export** (or you wrap the import to map a named export to `default`).
- A `Suspense` boundary must be somewhere **above** the lazy component.
- **Route-based splitting** (as above) is the best starting point because users naturally load one page at a time.
- Do not lazy load tiny components. The extra network request costs more than it saves.

**Demo idea:** Build the app, then open the browser's Network tab. Click between pages and watch a new small JavaScript file load the first time each page is visited.

### 3.4 Error boundaries

If a component throws an error while rendering, React unmounts the **entire tree** and the user sees a blank screen. An **error boundary** catches that error, shows a fallback UI, and keeps the rest of the app alive.

Error boundaries must be **class components**, since there is no hook equivalent yet:

```jsx
import { Component } from "react";

class ErrorBoundary extends Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };            // switch to fallback UI
  }

  componentDidCatch(error, info) {
    console.error("Caught by boundary:", error, info.componentStack); // log to a service
  }

  render() {
    if (this.state.hasError) {
      return <h2>Something went wrong. Please refresh the page.</h2>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <Dashboard />
</ErrorBoundary>
```

**What error boundaries do NOT catch:**

- Errors inside event handlers (use `try/catch`)
- Asynchronous code such as `setTimeout` or `fetch` callbacks (use `try/catch`, as in Part 2)
- Server-side rendering errors
- Errors thrown by the boundary itself

In practice, many teams use the `react-error-boundary` library, which provides a ready-made boundary with a reset function and works with function components.

**Placement tip:** Use several boundaries. One around the whole app as a safety net, and smaller ones around risky widgets (charts, third-party embeds) so one failure does not take down the page.

### 3.5 Client-side vs server-side rendering

| | **Client-Side Rendering (CSR)** | **Server-Side Rendering (SSR)** |
|---|---|---|
| Who builds the HTML? | The browser, using JavaScript | The server, per request |
| First response | Nearly empty HTML plus a JS bundle | Fully formed HTML |
| First paint | Slower (wait for JS, then fetch data) | Faster |
| After first load | Very fast navigation (no full reloads) | Needs **hydration** to become interactive |
| SEO | Weaker (crawlers may see little) | Stronger |
| Server cost | Low (static files and a CDN) | Higher (server renders each request) |
| Typical tools | Vite, Create React App | Next.js, Remix |
| Best for | Dashboards, internal tools, apps behind login | Marketing sites, blogs, e-commerce, public content |

**Hydration:** after the server sends HTML, React runs in the browser and attaches event handlers to that existing markup so it becomes interactive.

**Related terms (brief mention):** *Static Site Generation (SSG)* builds pages once at build time. *Incremental regeneration* rebuilds them periodically. Frameworks like Next.js let you mix all of these per page.

**What we have built so far in this course is CSR.** Everything in Parts 1 and 2 ran in the browser.

### 3.6 Other quick performance wins (optional extension)

- `React.memo` to skip re-rendering a component when its props have not changed.
- `useMemo` and `useCallback` to avoid recomputing values or recreating functions. Use them only after measuring a real problem.
- Stable, unique `key` values in lists.
- Virtualize very long lists (e.g. `react-window`).
- Always measure with React DevTools Profiler and Lighthouse before optimizing.

---

## Hands-On Exercise (20 to 25 minutes)

**Build a mini user directory.**

1. Create routes: `/` (Home), `/users` (list), `/users/:userId` (detail), and `*` (404). Use a shared `Layout` with a `NavLink` menu and an `Outlet`.
2. On `/users`, fetch `https://jsonplaceholder.typicode.com/users` and display the names, each linking to `/users/:userId`.
3. On the detail page, fetch `https://jsonplaceholder.typicode.com/users/:userId` using the `useFetch` hook. Show **loading**, **error**, and **success** states.
4. Lazy load the `Users` and `UserDetail` pages with `React.lazy` and `Suspense`.
5. Wrap the detail page in an `ErrorBoundary`.

**Stretch goals:**

- Add a search box that stores its text in the query string with `useSearchParams`.
- Make the detail page throw an error on purpose when `userId` is `999` to test the boundary.
- Try a user ID that does not exist (e.g. `/users/9999`) and handle the 404 response gracefully.

---

## Quick Recap

| Topic | One-line takeaway |
|---|---|
| Router | Maps URLs to components without reloading the page. |
| `Link` / `NavLink` | Navigate without reload. `NavLink` knows when it is active. |
| Dynamic and nested routes | `:param` plus `useParams`. Shared layouts with `Outlet`. |
| Fetch API | Check `res.ok` yourself, because `fetch` does not throw on 404 or 500. |
| Loading and error states | Every request has three states: loading, success, error. |
| Code splitting and lazy loading | Load code only when needed using `React.lazy` and `Suspense`. |
| Error boundaries | Class components that catch render errors and show a fallback. |
| CSR vs SSR | CSR renders in the browser. SSR renders HTML on the server for faster first paint and better SEO. |

## Check-for-Understanding Questions

1. What is the difference between `<a href="/about">` and `<Link to="/about">`?
2. In a nested route, what does `<Outlet />` render?
3. Why does `fetch("/missing")` not land in the `catch` block when the server returns 404?
4. What is the purpose of the cleanup function returned from `useEffect` in the fetch example?
5. Why must a lazy component be wrapped in `Suspense`?
6. Name two kinds of errors an error boundary will **not** catch.
7. A news website cares a lot about SEO. Would you pick CSR or SSR, and why?

## Reference Links

- React Router: [reactrouter.com](https://reactrouter.com)
- React docs on `lazy`, `Suspense`, and error boundaries: [react.dev](https://react.dev)
- MDN, Using the Fetch API: [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- The GeeksforGeeks articles from your syllabus list are good for student self-study after class.
