# React for Beginners: Routing, API Integration, and Performance

### A step-by-step guide where you build one small app from scratch

> **Who is this for?** Anyone who has learned the basics of React (components, props, `useState`) and now wants to build a real multi-page app that loads data from the internet.
>
> **How it works:** Instead of reading theory only, you will build **one project**, a *User Directory*, step by step. Every step tells you **what to do, what to type, and what you should see**.

---

## Table of Contents

0. [Before You Start](#0-before-you-start)
1. [Project Setup](#1-project-setup)
2. [Part 1: Routing (moving between pages)](#2-part-1-routing)
3. [Part 2: API Integration (loading data from the internet)](#3-part-2-api-integration)
4. [Part 3: UI & Performance (faster apps, safer apps)](#4-part-3-ui--performance)
5. [Final Project Structure](#5-final-project-structure)
6. [Practice Exercises](#6-practice-exercises)
7. [Common Errors and How to Fix Them](#7-common-errors-and-how-to-fix-them)
8. [Glossary](#8-glossary-simple-definitions)
9. [Quiz with Answers](#9-quiz-with-answers)
10. [Cheat Sheet](#10-cheat-sheet)
11. [Putting Your Project on GitHub](#11-putting-your-project-on-github)

---

## 0. Before You Start

### What you should already know

- Basic HTML, CSS, and JavaScript (variables, functions, arrays, `async/await` helps but is explained below).
- React basics: what a **component** is, what **props** are, and what `useState` does.

### Quick refresher (2 minutes)

**Component:** a JavaScript function that returns what should appear on screen.

```jsx
function Hello() {
  return <h1>Hello!</h1>;
}
```

**JSX:** the HTML-looking code inside JavaScript. Remember: write `className` (not `class`), and every component must start with a **capital letter**.

**`useState`:** gives a component a memory. When the value changes, the screen updates.

```jsx
const [count, setCount] = useState(0);   // count = current value, setCount = function to change it
```

**`useEffect`:** runs code *after* the component appears on screen. We use it later to load data.

```jsx
useEffect(() => {
  // code to run
}, []);   // [] means "run only once, when the component first appears"
```

### What you will build

A small app with:

| Page | Address (URL) | What it shows |
|---|---|---|
| Home | `/` | A welcome message |
| About | `/about` | Information about the app |
| Users | `/users` | A list of users loaded from the internet |
| User Detail | `/users/3` | Details of one user |
| Not Found | anything else | A friendly "page not found" message |

### Tools you need

1. **Node.js** (version 20 or newer). It lets you run JavaScript outside the browser and gives you `npm`.
2. **A code editor.** [VS Code](https://code.visualstudio.com) is recommended.
3. **A browser** (Chrome, Edge, or Firefox).
4. **Git** (only needed for the last section).

Check whether Node is installed. Open a terminal (in VS Code: **Terminal → New Terminal**) and type:

```bash
node --version
npm --version
```

You should see version numbers, such as `v20.11.0`. If you see "command not found", download Node.js (the **LTS** version) from [nodejs.org](https://nodejs.org), install it, then close and reopen your terminal.

---

## 1. Project Setup

### Step 1.1: Create the project

In your terminal, go to the folder where you keep projects, then run:

```bash
npm create vite@latest user-directory -- --template react
```

- **Vite** is a tool that creates and runs React projects very quickly.
- `user-directory` is the project name (a new folder will be created).
- `--template react` means "give me a React project".

If it asks extra questions, choose **React** and **JavaScript**.

### Step 1.2: Go into the folder and install packages

```bash
cd user-directory
npm install
```

`npm install` downloads all the libraries the project needs into a folder called `node_modules`. This can take a minute.

### Step 1.3: Install the router library

```bash
npm install react-router-dom
```

This adds **React Router**, the library that handles pages in React apps.

### Step 1.4: Start the app

```bash
npm run dev
```

The terminal prints a link like `http://localhost:5173`. Hold **Ctrl** (or **Cmd** on Mac) and click it, or paste it into your browser.

**Checkpoint:** You should see the Vite + React welcome page with a counter button. Leave this terminal running. It automatically refreshes the browser every time you save a file.

> [!TIP]
> To stop the server later, click in the terminal and press **Ctrl + C**. To start again, run `npm run dev`.

### Step 1.5: Clean up the starter files

The template comes with demo code we do not need.

1. Open the project in VS Code (**File → Open Folder → user-directory**).
2. Delete the file `src/App.css`.
3. Delete the folder `src/assets`.
4. Replace the entire contents of `src/index.css` with this simple style sheet:

```css
body {
  font-family: system-ui, sans-serif;
  margin: 0;
  color: #222;
  background: #fafafa;
}

nav {
  display: flex;
  gap: 16px;
  padding: 12px 24px;
  background: #1f2937;
}

nav a {
  color: #d1d5db;
  text-decoration: none;
  padding-bottom: 2px;
}

nav a.active {
  color: #ffffff;
  font-weight: bold;
  border-bottom: 2px solid #60a5fa;
}

main {
  max-width: 700px;
  margin: 24px auto;
  padding: 0 16px;
}

ul {
  line-height: 1.9;
}

.error {
  color: #b91c1c;
  background: #fee2e2;
  padding: 12px;
  border-radius: 6px;
}
```

5. Replace the entire contents of `src/App.jsx` with:

```jsx
export default function App() {
  return <h1>My User Directory</h1>;
}
```

**Checkpoint:** The browser now shows only the heading "My User Directory". Setup is done.

---

## 2. Part 1: Routing

### 2.1 What is routing? (simple explanation)

A normal website has many HTML files. Clicking a link loads a **new file** from the server, and the page flashes white.

A React app is different. It is a **Single Page Application (SPA)**: the browser loads **one** page, and JavaScript decides what to show.

The problem: if JavaScript swaps content but the address bar never changes, you get no back button, no bookmarks, and no shareable links.

**Routing** solves this. A **router** is like a receptionist:

> You say "I want `/about`." The receptionist looks at a list and says "That is the About component. Showing it now," and updates the address bar, **without reloading the page**.

### 2.2 Step by step: build the pages

#### Step 2.1: Create the page components

Inside `src`, create a new folder named `pages`. Inside it, create these files.

**`src/pages/Home.jsx`**

```jsx
export default function Home() {
  return (
    <div>
      <h1>Welcome</h1>
      <p>This is the home page of the User Directory.</p>
    </div>
  );
}
```

**`src/pages/About.jsx`**

```jsx
export default function About() {
  return (
    <div>
      <h1>About</h1>
      <p>This app was built to learn routing, API calls, and performance in React.</p>
    </div>
  );
}
```

**`src/pages/NotFound.jsx`**

```jsx
export default function NotFound() {
  return (
    <div>
      <h1>404: Page not found</h1>
      <p>Sorry, we could not find the page you were looking for.</p>
    </div>
  );
}
```

> **What does `export default` mean?** It makes the component available to other files. Other files then use `import Home from "./pages/Home"`.

#### Step 2.2: Turn on the router

Open `src/main.jsx` and replace everything with:

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import App from "./App.jsx";
import "./index.css";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

**What changed?** We wrapped `<App />` inside `<BrowserRouter>`. This gives every component in the app access to routing features. You do this **once**, at the top, and never touch it again.

#### Step 2.3: Define the routes

Open `src/App.jsx` and replace everything with:

```jsx
import { Routes, Route } from "react-router-dom";
import Home from "./pages/Home";
import About from "./pages/About";
import NotFound from "./pages/NotFound";

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
      <Route path="*" element={<NotFound />} />
    </Routes>
  );
}
```

**Line by line:**

| Code | Meaning |
|---|---|
| `<Routes>` | A container. It looks at the current URL and picks **one** matching route. |
| `<Route path="/" element={<Home />} />` | "If the URL is `/`, show the Home component." |
| `<Route path="/about" ... />` | "If the URL is `/about`, show About." |
| `path="*"` | A wildcard that means "anything else". This is how we make a 404 page. |

**Checkpoint:** In the browser address bar, type `http://localhost:5173/about`. You should see the About page. Now try `/hello`. You should see the 404 page. Routing works.

But there is no way to click between pages yet. Let's fix that.

#### Step 2.4: Add navigation links

> [!WARNING]
> **Never use a normal `<a href="/about">` for pages inside your app.** It makes the browser reload everything, and your React state is lost. Use `Link` instead.

We want a menu that appears on **every** page, so we create a shared layout.

Create a folder `src/components`, then create **`src/components/Layout.jsx`**:

```jsx
import { Link, Outlet } from "react-router-dom";

export default function Layout() {
  return (
    <>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
      </nav>

      <main>
        <Outlet />
      </main>
    </>
  );
}
```

**New idea: `<Outlet />`.** It is a placeholder that means "the selected page appears **here**". The `nav` stays the same while only the content inside `main` changes.

Now tell `App.jsx` to use this layout. Replace `App.jsx` with:

```jsx
import { Routes, Route } from "react-router-dom";
import Layout from "./components/Layout";
import Home from "./pages/Home";
import About from "./pages/About";
import NotFound from "./pages/NotFound";

export default function App() {
  return (
    <Routes>
      <Route element={<Layout />}>
        <Route index element={<Home />} />
        <Route path="about" element={<About />} />
        <Route path="*" element={<NotFound />} />
      </Route>
    </Routes>
  );
}
```

**What changed?**

- The pages are now **nested inside** a parent `<Route element={<Layout />}>`. The parent has no `path`, so it only provides the shared layout.
- `index` means "the default child", shown when the URL is just `/`.
- Child paths are **relative**, so we write `about`, not `/about`.

**Checkpoint:** You now see a dark menu bar at the top. Click **About**, then **Home**. The page changes instantly, the address bar updates, and the browser's back button works. Open DevTools (**F12**), go to the **Network** tab, and click the links again. Notice there is **no full page reload**.

#### Step 2.5: Highlight the current page with `NavLink`

`NavLink` is a `Link` that knows whether its page is the current one. It automatically adds a CSS class called `active` (we already styled it in `index.css`).

In `Layout.jsx`, change the import and the links:

```jsx
import { NavLink, Outlet } from "react-router-dom";

export default function Layout() {
  return (
    <>
      <nav>
        <NavLink to="/" end>Home</NavLink>
        <NavLink to="/about">About</NavLink>
        <NavLink to="/users">Users</NavLink>
      </nav>

      <main>
        <Outlet />
      </main>
    </>
  );
}
```

**Why `end` on Home?** Every URL starts with `/`. Without `end`, Home would look "active" on every page. `end` means "only match if the URL is exactly `/`".

**Checkpoint:** The current page's link is now white, bold, and underlined in blue. (The Users link leads to a 404 for now. We build that next.)

#### Step 2.6: Create the Users list page

Create **`src/pages/Users.jsx`**. For now we use a hard-coded list. In Part 2 we will load real data.

```jsx
import { Link } from "react-router-dom";

const users = [
  { id: 1, name: "Leanne Graham" },
  { id: 2, name: "Ervin Howell" },
  { id: 3, name: "Clementine Bauch" },
];

export default function Users() {
  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map((user) => (
          <li key={user.id}>
            <Link to={`/users/${user.id}`}>{user.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

> **What is `key`?** When you render a list, React needs a unique `key` on each item so it can track them. Use the item's `id`.

> **What are the backticks?** `` `/users/${user.id}` `` is a *template string*. If `user.id` is 3, the result is `/users/3`.

Now add the route. In `App.jsx`, add an import and a route:

```jsx
import Users from "./pages/Users";
```

```jsx
<Route path="users" element={<Users />} />
```

Place the new `<Route>` line right after the `about` route. The result should look like this:

```jsx
<Route element={<Layout />}>
  <Route index element={<Home />} />
  <Route path="about" element={<About />} />
  <Route path="users" element={<Users />} />
  <Route path="*" element={<NotFound />} />
</Route>
```

**Checkpoint:** Click **Users**. You see three names. Clicking a name changes the URL to `/users/1`, but shows the 404 page, because we have not made that route yet.

#### Step 2.7: Dynamic routes (one page for many users)

We do not want to create a separate route for every user. Instead we use a **URL parameter**, a part of the path that changes. We write it with a colon: `:userId`.

Create **`src/pages/UserDetail.jsx`**:

```jsx
import { useParams, Link } from "react-router-dom";

export default function UserDetail() {
  const { userId } = useParams();

  return (
    <div>
      <h1>User #{userId}</h1>
      <p>Details for user {userId} will appear here soon.</p>
      <Link to="/users">&larr; Back to users</Link>
    </div>
  );
}
```

**What does `useParams()` do?** It reads the changing part of the URL. If the URL is `/users/3`, then `userId` is `"3"`.

> [!NOTE]
> URL parameters are always **text (strings)**, even if they look like numbers. If you need a real number, use `Number(userId)`.

Add the import and route in `App.jsx`:

```jsx
import UserDetail from "./pages/UserDetail";
```

```jsx
<Route path="users/:userId" element={<UserDetail />} />
```

Your routes now look like:

```jsx
<Route element={<Layout />}>
  <Route index element={<Home />} />
  <Route path="about" element={<About />} />
  <Route path="users" element={<Users />} />
  <Route path="users/:userId" element={<UserDetail />} />
  <Route path="*" element={<NotFound />} />
</Route>
```

**Checkpoint:** Go to Users and click any name. You see "User #3" (or whichever you clicked). Manually type `/users/42` in the address bar. It shows "User #42". One route handles every user.

#### Step 2.8: Navigate with code using `useNavigate`

Sometimes you need to change pages **without** the user clicking a link, for example after a form is submitted. Use the `useNavigate` hook.

Add a button to `UserDetail.jsx`. Update the file:

```jsx
import { useParams, Link, useNavigate } from "react-router-dom";

export default function UserDetail() {
  const { userId } = useParams();
  const navigate = useNavigate();

  return (
    <div>
      <h1>User #{userId}</h1>
      <p>Details for user {userId} will appear here soon.</p>
      <Link to="/users">&larr; Back to users</Link>
      <br /><br />
      <button onClick={() => navigate(-1)}>Go back (previous page)</button>
    </div>
  );
}
```

`navigate(-1)` works like the browser's back button. `navigate("/about")` jumps to a specific page. Adding `{ replace: true }` (for example `navigate("/", { replace: true })`) swaps the current history entry instead of adding a new one.

### 2.3 Extra routing tools (reference)

#### Router types

| Router | URL looks like | When to use it |
|---|---|---|
| `BrowserRouter` | `/about` | **Default for websites.** Clean URLs. |
| `HashRouter` | `/#/about` | Hosting that cannot be configured for deep links. |
| `MemoryRouter` | none shown | Tests and non-browser apps. |
| `StaticRouter` | set by server | Server-side rendering. |

#### Router hooks

| Hook | What it does | Example |
|---|---|---|
| `useParams()` | Reads URL parameters | `/users/:userId` gives `userId` |
| `useNavigate()` | Changes page from code | After login, go to dashboard |
| `useLocation()` | Tells you the current URL details | Highlight a menu, track page views |
| `useSearchParams()` | Reads and writes `?key=value` in the URL | Filters like `/users?sort=asc` |

#### Bonus: query strings with `useSearchParams`

```jsx
import { useSearchParams } from "react-router-dom";

function SortButton() {
  const [params, setParams] = useSearchParams();
  const sort = params.get("sort") ?? "asc";

  return (
    <button onClick={() => setParams({ sort: sort === "asc" ? "desc" : "asc" })}>
      Sorting: {sort}
    </button>
  );
}
```

Clicking changes the address bar to `?sort=desc`. Because the state lives in the URL, the user can bookmark or share it.

#### Bonus: protecting a page (redirects)

```jsx
import { Navigate } from "react-router-dom";

function ProtectedRoute({ isLoggedIn, children }) {
  if (!isLoggedIn) {
    return <Navigate to="/login" replace />;
  }
  return children;
}
```

Use it by wrapping a page: `<ProtectedRoute isLoggedIn={user}><Dashboard /></ProtectedRoute>`. Where does `isLoggedIn` come from? Usually from Context or Redux (last topic).

### 2.4 Part 1 recap

- A **router** matches URLs to components without reloading.
- `BrowserRouter` goes once at the top. `Routes` + `Route` define pages.
- `Link` / `NavLink` replace `<a>`. `NavLink` marks the active page.
- `Outlet` shows the child page inside a shared layout.
- `:param` + `useParams()` gives dynamic pages.
- `path="*"` catches everything else (404).

---

## 3. Part 2: API Integration

### 3.1 What is an API? (simple explanation)

Think of a restaurant:

- **You** are the React app.
- **The kitchen** is the server that holds the data.
- **The waiter** is the **API**. You give the waiter a request ("one list of users, please"), and the waiter brings back a response.

In practice, an API is just a web address that returns **data** (usually in JSON format) instead of a web page. Open this link in your browser to see one:

`https://jsonplaceholder.typicode.com/users`

You will see a long list of fake users. JSONPlaceholder is a free practice API, so we can use it safely.

### 3.2 HTTP requests: the five verbs

When your app talks to an API, it picks a **method** that says what it wants to do:

| Method | Meaning | Real-life example |
|---|---|---|
| `GET` | **Read** data | Show me all users |
| `POST` | **Create** new data | Add a new user |
| `PUT` | **Replace** existing data | Replace this user's full record |
| `PATCH` | **Change part** of data | Update only this user's email |
| `DELETE` | **Remove** data | Delete this user |

The server replies with a **status code**:

| Code | Meaning |
|---|---|
| `200` | OK, success |
| `201` | Created successfully |
| `400` | Bad request (you sent something wrong) |
| `401` / `403` | Not logged in / not allowed |
| `404` | Not found |
| `500` | The server had a problem |

### 3.3 The most important idea: three states

Loading data takes time (maybe half a second, maybe never if the internet drops). Your screen must handle **three situations**:

```
 Request starts          Request succeeds
 ┌──────────────┐        ┌──────────────┐
 │   LOADING    │ -----> │   SUCCESS    │   show the data
 │ "Please wait"│        └──────────────┘
 └──────┬───────┘
        │  Request fails
        ▼
 ┌──────────────┐
 │    ERROR     │   show a helpful message
 └──────────────┘
```

Beginners usually code only the success case, and then the app shows a blank page or crashes whenever something goes wrong. We will handle all three.

So we need three pieces of state:

```jsx
const [users, setUsers] = useState([]);      // the data
const [loading, setLoading] = useState(true); // are we still waiting?
const [error, setError] = useState(null);     // did something go wrong?
```

### 3.4 Step by step: load the Users list from the API

Replace the entire contents of **`src/pages/Users.jsx`** with:

```jsx
import { useState, useEffect } from "react";
import { Link } from "react-router-dom";

export default function Users() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function loadUsers() {
      try {
        const response = await fetch("https://jsonplaceholder.typicode.com/users");

        if (!response.ok) {
          throw new Error(`Request failed with status ${response.status}`);
        }

        const data = await response.json();
        setUsers(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    loadUsers();
  }, []);

  if (loading) return <p>Loading users...</p>;
  if (error) return <p className="error">Something went wrong: {error}</p>;

  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map((user) => (
          <li key={user.id}>
            <Link to={`/users/${user.id}`}>{user.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**Checkpoint:** Click **Users**. You briefly see "Loading users..." and then **ten** real names from the internet.

#### Understanding every part

| Code | Plain-English meaning |
|---|---|
| `useState(true)` for `loading` | At the start we are waiting, so loading is `true`. |
| `useEffect(() => {...}, [])` | "After the page first appears, run this code once." |
| `async function loadUsers()` | A function that can **wait** for slow things. |
| `await fetch(url)` | Send the request and **pause here** until the server answers. |
| `response.ok` | `true` when the status is 200 to 299. |
| `throw new Error(...)` | Deliberately trigger the error path. |
| `await response.json()` | Convert the reply text into a JavaScript object or array. |
| `setUsers(data)` | Save the data. React re-renders and shows the list. |
| `catch (err)` | Runs if anything above fails. We save the message. |
| `finally { setLoading(false) }` | Runs **always**, success or failure, so the "Loading..." message disappears. |
| `if (loading) return ...` | Show the loading message **instead of** the normal page. |

> [!WARNING]
> **The #1 beginner trap:** `fetch` does **not** treat 404 or 500 as errors. It only fails when the network itself fails. That is why we check `response.ok` and throw our own error.

> [!NOTE]
> **Why did my request run twice?** In development, `StrictMode` (in `main.jsx`) runs effects twice on purpose to help find bugs. You may see two requests in the Network tab. It does **not** happen in the final production build.

### 3.5 Test the error state (very important!)

A good developer tests failure, not just success. Try both:

**Test A: a wrong URL.** In `Users.jsx`, temporarily change `/users` to `/userz` in the fetch address and save. You should see the red message: *"Something went wrong: Request failed with status 404"*. Change it back.

**Test B: no internet.** Press **F12**, open the **Network** tab, set the throttling dropdown (it says "No throttling") to **Offline**, and click Users again. You should see *"Something went wrong: Failed to fetch"*. Set it back to "No throttling".

Also try the **Slow 4G** setting, which makes the "Loading..." message stay visible longer.

### 3.6 Step by step: the User Detail page with real data

Now replace **`src/pages/UserDetail.jsx`** with a version that loads one user:

```jsx
import { useState, useEffect } from "react";
import { useParams, Link } from "react-router-dom";

export default function UserDetail() {
  const { userId } = useParams();
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function loadUser() {
      try {
        setLoading(true);
        setError(null);

        const response = await fetch(
          `https://jsonplaceholder.typicode.com/users/${userId}`
        );
        if (!response.ok) {
          throw new Error(`Request failed with status ${response.status}`);
        }

        setUser(await response.json());
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    loadUser();
  }, [userId]);

  if (loading) return <p>Loading user...</p>;
  if (error) return <p className="error">Something went wrong: {error}</p>;

  return (
    <div>
      <h1>{user.name}</h1>
      <p><strong>Username:</strong> {user.username}</p>
      <p><strong>Email:</strong> {user.email}</p>
      <p><strong>Phone:</strong> {user.phone}</p>
      <p><strong>City:</strong> {user.address.city}</p>
      <p><strong>Company:</strong> {user.company.name}</p>
      <Link to="/users">&larr; Back to users</Link>
    </div>
  );
}
```

**One new idea: the dependency array `[userId]`.** Previously we wrote `[]` ("run once"). Now we write `[userId]`, which means "run again **whenever `userId` changes**". So going from `/users/1` to `/users/2` automatically loads the new user.

**Checkpoint:** Open Users, click a name, and see the full details. Now type `/users/9999` in the address bar. You should see the 404 error message, because that user does not exist.

### 3.7 Clean up: build your own reusable hook

Look at `Users.jsx` and `UserDetail.jsx`. The loading, error, and data logic is **almost identical**. Copy-pasting code is a sign we should create a **custom hook**, a function that starts with `use` and packages reusable logic.

Create a folder `src/hooks`, then create **`src/hooks/useFetch.js`**:

```jsx
import { useState, useEffect } from "react";

export default function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    async function load() {
      try {
        setLoading(true);
        setError(null);

        const response = await fetch(url, { signal: controller.signal });
        if (!response.ok) {
          throw new Error(`Request failed with status ${response.status}`);
        }

        setData(await response.json());
      } catch (err) {
        if (err.name !== "AbortError") {
          setError(err.message);
        }
      } finally {
        setLoading(false);
      }
    }

    load();

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}
```

**The new piece: `AbortController`.** Imagine you click a user and immediately click another one. The first request is now useless. `controller.abort()` cancels it. React runs the function we `return` from `useEffect` (called the **cleanup function**) when the page changes or the URL changes, so old requests never overwrite newer ones.

Now simplify both pages.

**`src/pages/Users.jsx`:**

```jsx
import { Link } from "react-router-dom";
import useFetch from "../hooks/useFetch";

export default function Users() {
  const { data: users, loading, error } = useFetch(
    "https://jsonplaceholder.typicode.com/users"
  );

  if (loading) return <p>Loading users...</p>;
  if (error) return <p className="error">Something went wrong: {error}</p>;

  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map((user) => (
          <li key={user.id}>
            <Link to={`/users/${user.id}`}>{user.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**`src/pages/UserDetail.jsx`:**

```jsx
import { useParams, Link } from "react-router-dom";
import useFetch from "../hooks/useFetch";

export default function UserDetail() {
  const { userId } = useParams();
  const { data: user, loading, error } = useFetch(
    `https://jsonplaceholder.typicode.com/users/${userId}`
  );

  if (loading) return <p>Loading user...</p>;
  if (error) return <p className="error">Something went wrong: {error}</p>;

  return (
    <div>
      <h1>{user.name}</h1>
      <p><strong>Username:</strong> {user.username}</p>
      <p><strong>Email:</strong> {user.email}</p>
      <p><strong>Phone:</strong> {user.phone}</p>
      <p><strong>City:</strong> {user.address.city}</p>
      <p><strong>Company:</strong> {user.company.name}</p>
      <Link to="/users">&larr; Back to users</Link>
    </div>
  );
}
```

> The syntax `data: users` means "take the `data` value and call it `users` here". It is called *destructuring with renaming*.

**Checkpoint:** The app behaves exactly the same as before, but each page is much shorter. That is a successful refactor.

### 3.8 Optional: sending data with POST

So far we only **read** data. To **create** data, send a `POST` request with a body. Here is a function (do not worry about adding it to the app unless you want to practice):

```jsx
async function addUser(newUser) {
  const response = await fetch("https://jsonplaceholder.typicode.com/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(newUser),
  });

  if (!response.ok) {
    throw new Error("Could not create user");
  }
  return response.json();
}

// Try it:
addUser({ name: "Your Name", email: "you@example.com" }).then(console.log);
```

- `method: "POST"` says we are creating something.
- `headers` tells the server the body is JSON.
- `JSON.stringify` turns a JavaScript object into text for sending.

> [!NOTE]
> JSONPlaceholder only **pretends** to save data. It replies with success but does not store anything. That makes it perfect and safe for practice.

### 3.9 Good error handling checklist

| What can go wrong | What the app should do |
|---|---|
| Internet is down | Show "Check your connection" and, ideally, a **Retry** button |
| Server says 404 | Show "We could not find that" |
| Server says 500 | Show "The server had a problem, try again later" |
| Data is empty | Show "No users yet" instead of a blank page |
| Slow response | Show a loading message or a spinner |

### 3.10 Going further

Writing fetch logic by hand is the best way to learn, but in big apps people use libraries that add caching, automatic retries, and refresh. The most popular are **TanStack Query** and **SWR**. If you used Redux Toolkit, **RTK Query** does the same job. Learn the manual way first (what you just did), and then these libraries will make complete sense.

### 3.11 Part 2 recap

- An API is a web address that returns **data**. Use `fetch` to call it.
- Every request has **three states**: loading, success, error.
- Check `response.ok`, because `fetch` does not throw on 404/500.
- `useEffect` triggers the request. The dependency array controls **when** it re-runs.
- A cleanup function with `AbortController` cancels outdated requests.
- Repeating logic? Extract a **custom hook** like `useFetch`.

---

## 4. Part 3: UI & Performance

### 4.1 Why does performance matter?

Right now, when someone visits your site, the browser downloads **all** the code for **every** page, even pages they never open. For our tiny app that is fine. For a big app (thousands of components), the first load becomes slow, especially on mobile data.

**Analogy:** Imagine you ask for a single pizza, and the restaurant also delivers the entire menu, the whole kitchen, and ingredients for 100 other dishes. It is better to deliver only what you ordered, and the rest **on demand**.

### 4.2 Code splitting and lazy loading

- **Code splitting:** breaking one big JavaScript file into several smaller files (called **chunks**).
- **Lazy loading:** downloading a chunk only at the moment it is needed.

React provides `lazy` and `Suspense` for this.

- `lazy(() => import("./SomeComponent"))`: "Don't load this file until someone actually needs it."
- `<Suspense fallback={...}>`: "While the file downloads, show this fallback message."

### 4.3 Step by step: lazy load your pages

#### Step 4.1: Update `App.jsx`

Replace **`src/App.jsx`** with:

```jsx
import { lazy } from "react";
import { Routes, Route } from "react-router-dom";
import Layout from "./components/Layout";
import Home from "./pages/Home";
import NotFound from "./pages/NotFound";

const About = lazy(() => import("./pages/About"));
const Users = lazy(() => import("./pages/Users"));
const UserDetail = lazy(() => import("./pages/UserDetail"));

export default function App() {
  return (
    <Routes>
      <Route element={<Layout />}>
        <Route index element={<Home />} />
        <Route path="about" element={<About />} />
        <Route path="users" element={<Users />} />
        <Route path="users/:userId" element={<UserDetail />} />
        <Route path="*" element={<NotFound />} />
      </Route>
    </Routes>
  );
}
```

**What changed?** `About`, `Users`, and `UserDetail` are no longer imported at the top. They are created with `lazy(...)`. We keep `Home` and `NotFound` as normal imports because they are tiny and needed straight away.

#### Step 4.2: Add `Suspense` to the layout

A lazy component needs a `Suspense` **above** it. We place it around the `Outlet`, so the menu stays visible while a page downloads.

Replace **`src/components/Layout.jsx`** with:

```jsx
import { Suspense } from "react";
import { NavLink, Outlet } from "react-router-dom";

export default function Layout() {
  return (
    <>
      <nav>
        <NavLink to="/" end>Home</NavLink>
        <NavLink to="/about">About</NavLink>
        <NavLink to="/users">Users</NavLink>
      </nav>

      <main>
        <Suspense fallback={<p>Loading page...</p>}>
          <Outlet />
        </Suspense>
      </main>
    </>
  );
}
```

#### Step 4.3: See it working

1. Open the app in your browser and press **F12**.
2. Go to the **Network** tab and click the **JS** filter.
3. Clear the list (the circle with a line through it).
4. Click **About**.

**Checkpoint:** A small new file (named something like `About.jsx` in dev mode) appears in the list the first time you visit that page. You just proved that the code downloads **only when needed**.

> [!TIP]
> To feel the loading message, set the Network throttling to **Slow 4G** first.

**Rules for lazy loading:**

1. The lazy component must use `export default`.
2. A `Suspense` must exist somewhere **above** it.
3. Lazy loading **pages** (routes) is the best place to start.
4. Do **not** lazy-load tiny components. The extra network request costs more than it saves.

### 4.4 Error boundaries: stop the blank screen

#### The problem

If a component crashes **while rendering** (for example, it reads a value that does not exist), React removes the **entire** app and the user sees a **blank white screen**. That is terrible.

An **error boundary** is a safety net. It catches the crash, shows a friendly message, and keeps the rest of the app working.

> [!NOTE]
> Error boundaries must be written as **class components**. React does not yet offer a hook version. You only write this one once, so don't worry about learning classes in depth.

#### Step 4.4: Create the error boundary

Create **`src/components/ErrorBoundary.jsx`**:

```jsx
import { Component } from "react";

export default class ErrorBoundary extends Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    console.error("ErrorBoundary caught:", error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error">
          <h2>Oops! Something went wrong.</h2>
          <p>Please go back and try again.</p>
        </div>
      );
    }
    return this.props.children;
  }
}
```

**How it works:**

| Part | Meaning |
|---|---|
| `state = { hasError: false }` | At first, no error. |
| `getDerivedStateFromError()` | React calls this when a child crashes. We flip `hasError` to `true`. |
| `componentDidCatch()` | A place to **log** the error (real apps send it to a monitoring service). |
| `render()` | If there was an error, show the fallback. Otherwise show the normal children. |

#### Step 4.5: Wrap a page with it

In **`App.jsx`**, add the import:

```jsx
import ErrorBoundary from "./components/ErrorBoundary";
```

Then change the user detail route to:

```jsx
<Route
  path="users/:userId"
  element={
    <ErrorBoundary>
      <UserDetail />
    </ErrorBoundary>
  }
/>
```

#### Step 4.6: Test it with a deliberate crash

In **`UserDetail.jsx`**, add this line **right after** the `useFetch` call (before the `if (loading)` line):

```jsx
if (userId === "999") {
  throw new Error("Test crash!");
}
```

Now visit `http://localhost:5173/users/999`.

**Checkpoint:** You should see the red "Oops! Something went wrong." box **inside the page**, and the navigation menu **still works**. Click **Users** to leave the error page. (In development mode, Vite may also show a red error overlay on top. Press **Esc** or click the X to close it. It does not appear in the final production app.)

Remove the test lines afterwards.

#### What error boundaries do **not** catch

| Not caught | What to use instead |
|---|---|
| Errors inside **event handlers** (e.g. `onClick`) | `try / catch` |
| Errors in **async code** (`fetch`, `setTimeout`) | `try / catch` (as we did in Part 2) |
| Errors in server-side rendering | Server error handling |
| Errors inside the boundary itself | A boundary higher up |

**Where to place them:** Use one around the whole app as a last safety net, and smaller ones around risky parts (charts, embedded widgets) so one failure does not break the whole page.

*Tip: the popular `react-error-boundary` library gives you a ready-made boundary that works with function components and has a "try again" feature.*

### 4.5 Client-side vs server-side rendering

Where is the HTML of a page **built**?

**Client-Side Rendering (CSR): what our app does**

1. The browser asks for the page.
2. The server sends an almost **empty** HTML file plus a big JavaScript bundle.
3. The browser runs the JavaScript, which **builds the page** and fetches the data.

**Server-Side Rendering (SSR)**

1. The browser asks for the page.
2. The server **builds the full HTML** (including data) and sends it.
3. The user sees content immediately. Then React "wakes up" the page in the browser, which is called **hydration**.

| | **CSR** | **SSR** |
|---|---|---|
| HTML is built by | The browser | The server |
| First screen appears | Slower (wait for JS and data) | Faster |
| Moving between pages | Very fast | Fast |
| Search-engine friendliness (SEO) | Weaker | Stronger |
| Server cost | Low (just static files) | Higher |
| Tools | Vite (what we used), Create React App | Next.js, Remix |
| Best for | Dashboards, admin panels, apps behind a login | Blogs, shops, news, marketing pages |

**Related terms:** *Static Site Generation (SSG)* builds pages once, ahead of time, when you deploy. Frameworks like **Next.js** let you choose CSR, SSR, or SSG for each page.

**Simple rule of thumb:** If search engines or fast first impressions matter (public content), consider SSR/SSG. If users must log in anyway, CSR is perfectly fine.

### 4.6 Bonus: a few more easy speed-ups

- `React.memo`: skips re-rendering a component if its props did not change.
- `useMemo` / `useCallback`: avoid repeating heavy work. **Use them only after you measure a real problem**, not by default.
- Always give list items stable, unique `key` values.
- Very long lists (thousands of rows)? Use a virtual list library such as `react-window`.
- **Measure before optimizing**: use React DevTools Profiler and Chrome's Lighthouse tab.

### 4.7 Part 3 recap

- **Code splitting** makes small chunks. **Lazy loading** fetches them on demand.
- `lazy()` + `Suspense` is how React does it. Start with routes.
- An **error boundary** prevents a crash from blanking the whole app.
- **CSR** builds pages in the browser, **SSR** builds them on the server.

---

## 5. Final Project Structure

After all steps, your `src` folder should look like this:

```
user-directory/
├── index.html
├── package.json
└── src/
    ├── main.jsx                  <- BrowserRouter wraps the app
    ├── App.jsx                   <- all routes + lazy loading
    ├── index.css                 <- styles
    ├── components/
    │   ├── Layout.jsx            <- nav + Suspense + Outlet
    │   └── ErrorBoundary.jsx     <- crash safety net
    ├── hooks/
    │   └── useFetch.js           <- reusable data-loading hook
    └── pages/
        ├── Home.jsx
        ├── About.jsx
        ├── Users.jsx             <- list, uses useFetch
        ├── UserDetail.jsx        <- one user, uses useFetch + useParams
        └── NotFound.jsx
```

**Final `src/App.jsx` for reference:**

```jsx
import { lazy } from "react";
import { Routes, Route } from "react-router-dom";
import Layout from "./components/Layout";
import ErrorBoundary from "./components/ErrorBoundary";
import Home from "./pages/Home";
import NotFound from "./pages/NotFound";

const About = lazy(() => import("./pages/About"));
const Users = lazy(() => import("./pages/Users"));
const UserDetail = lazy(() => import("./pages/UserDetail"));

export default function App() {
  return (
    <Routes>
      <Route element={<Layout />}>
        <Route index element={<Home />} />
        <Route path="about" element={<About />} />
        <Route path="users" element={<Users />} />
        <Route
          path="users/:userId"
          element={
            <ErrorBoundary>
              <UserDetail />
            </ErrorBoundary>
          }
        />
        <Route path="*" element={<NotFound />} />
      </Route>
    </Routes>
  );
}
```

---

## 6. Practice Exercises

Try these without looking at the steps first. Hints are provided.

### Easy

1. **Add a "Contact" page** at `/contact`.
   *Hint: create `Contact.jsx`, add a `<Route>` and a `<NavLink>`.*
2. **Show a message when the users list is empty.**
   *Hint: `if (users.length === 0) return <p>No users found.</p>;`*
3. **Add a Retry button** to the error message that reloads the page.
   *Hint: `<button onClick={() => window.location.reload()}>Retry</button>`*

### Medium

4. **Add a search box on the Users page** that filters names as you type.
   *Hint: store the typed text in `useState`, then use `users.filter(u => u.name.toLowerCase().includes(text.toLowerCase()))`.*
5. **Put the search text in the URL** (`/users?search=ann`) using `useSearchParams`.
   *Hint: replace `useState` with `const [params, setParams] = useSearchParams()`.*
6. **Create a "Posts" feature**: a `/posts` page listing posts from `https://jsonplaceholder.typicode.com/posts`, using `useFetch`.

### Challenge

7. **Show a user's posts on their detail page**, using `https://jsonplaceholder.typicode.com/posts?userId=3`.
8. **Build a protected `/admin` page** that redirects to `/` unless a "logged in" button was clicked. Store the logged-in state with Context.
9. **Run a production build** and look at the chunk files: `npm run build`, then `npm run preview`. How many JavaScript files were created? Why?

---

## 7. Common Errors and How to Fix Them

| What you see | Likely cause | Fix |
|---|---|---|
| Blank white page | A crash while rendering | Open the **Console** tab (F12) and read the red message |
| `useRoutes() may be used only in the context of a <Router>` | Forgot `BrowserRouter` | Wrap `<App />` in `main.jsx` |
| Every URL shows 404 | Missing or wrong `path` | Check spelling and slashes. Child routes should have **no** leading `/` |
| Page content does not appear in a nested route | Missing `<Outlet />` | Add `<Outlet />` in the layout component |
| Whole page reloads on every click | Used `<a href>` | Use `<Link>` or `<NavLink>` |
| `Cannot read properties of null (reading 'name')` | Using `user.name` before data arrived | Return early when `loading` is true or `data` is null |
| `Cannot read properties of undefined (reading 'map')` | `users` is not an array yet | Use the loading check first, or start with `useState([])` |
| Infinite loop of requests | Missing dependency array on `useEffect` | Add `[]` (or the right list) as the 2nd argument |
| Error says "Element type is invalid... default export" | Missing `export default` | Add `export default` before the component |
| `A component suspended while responding to synchronous input` | Lazy component without `Suspense` | Wrap it in `<Suspense>` |
| Deep link works locally but gives 404 after deploying | Host does not send `index.html` for all paths | Configure the host to redirect all routes to `index.html` |
| Changes do not show in the browser | Dev server stopped or file not saved | Save the file and check that `npm run dev` is still running |

---

## 8. Glossary (simple definitions)

| Term | Simple meaning |
|---|---|
| **SPA** | Single Page Application: one HTML page, with JavaScript swapping the content |
| **Router** | Code that matches the URL to the component to show |
| **Route** | One rule: "this path shows this component" |
| **Layout** | A shared frame (like a menu) around many pages |
| **Outlet** | The spot in a layout where the child page appears |
| **URL parameter** | A changing part of the path, like the `3` in `/users/3` |
| **Query string** | The `?key=value` part of a URL |
| **API** | A web address that sends and receives data |
| **JSON** | A text format for data that looks like JavaScript objects |
| **fetch** | Browser function that sends an HTTP request |
| **Promise / async / await** | A way to write code that waits for slow tasks (like network calls) |
| **HTTP method** | The action requested: GET, POST, PUT, PATCH, DELETE |
| **Status code** | A number in the server's reply: 200 good, 404 not found, 500 server error |
| **Side effect** | Work outside rendering (fetching data, timers). Done inside `useEffect` |
| **Dependency array** | The list at the end of `useEffect` that controls when it re-runs |
| **Cleanup function** | The function returned from `useEffect`, run when the component leaves |
| **Custom hook** | Your own function starting with `use` that reuses React logic |
| **Bundle** | The final JavaScript file(s) the browser downloads |
| **Code splitting** | Breaking a bundle into smaller chunks |
| **Lazy loading** | Downloading code only when it is needed |
| **Suspense** | A React component that shows a fallback while something loads |
| **Error boundary** | A component that catches crashes in its children and shows a fallback |
| **CSR** | Client-Side Rendering: the browser builds the page |
| **SSR** | Server-Side Rendering: the server builds the page |
| **Hydration** | React attaching interactivity to server-rendered HTML |

---

## 9. Quiz with Answers

Click each question to reveal the answer.

**1. What is the difference between `<a href="/about">` and `<Link to="/about">`?**

<details>
<summary>Show answer</summary>

`<a>` makes the browser reload the whole page, which resets all React state. `<Link>` changes the URL and swaps the component **without** reloading.
</details>

**2. What does `<Outlet />` do?**

<details>
<summary>Show answer</summary>

It marks the place inside a parent layout where the matching **child route's** component is displayed.
</details>

**3. If the URL is `/users/7` and the route is `users/:userId`, what does `useParams()` return?**

<details>
<summary>Show answer</summary>

`{ userId: "7" }`. Note that `"7"` is a string, not a number.
</details>

**4. Why do we check `response.ok` after calling `fetch`?**

<details>
<summary>Show answer</summary>

`fetch` only fails on network problems. A 404 or 500 reply is still a "successful" fetch, so we check `response.ok` and throw our own error.
</details>

**5. What are the three states every data request should handle?**

<details>
<summary>Show answer</summary>

Loading, success, and error.
</details>

**6. What does an empty dependency array `[]` mean in `useEffect`?**

<details>
<summary>Show answer</summary>

Run the effect **once**, after the component first appears.
</details>

**7. What is the purpose of the function returned from `useEffect` in `useFetch`?**

<details>
<summary>Show answer</summary>

It is the cleanup function. It cancels the request (`controller.abort()`) when the component leaves or the URL changes, so outdated responses are ignored.
</details>

**8. Why must lazy components be inside `<Suspense>`?**

<details>
<summary>Show answer</summary>

While the lazy file is downloading, React needs something to display. `Suspense` provides that fallback.
</details>

**9. Name two kinds of errors an error boundary will NOT catch.**

<details>
<summary>Show answer</summary>

Any two of: errors in event handlers, errors in async code (fetch, setTimeout), server-side rendering errors, errors inside the boundary itself.
</details>

**10. A news website needs to appear well in Google search. CSR or SSR?**

<details>
<summary>Show answer</summary>

SSR (or SSG), because the server sends complete HTML that search engines can read easily and users see content faster.
</details>

---

## 10. Cheat Sheet

### Routing

```jsx
// main.jsx
<BrowserRouter><App /></BrowserRouter>

// App.jsx
<Routes>
  <Route element={<Layout />}>              {/* shared layout */}
    <Route index element={<Home />} />      {/* "/" */}
    <Route path="users" element={<Users />} />
    <Route path="users/:id" element={<UserDetail />} />
    <Route path="*" element={<NotFound />} />
  </Route>
</Routes>

// Layout.jsx
<NavLink to="/users">Users</NavLink>
<Outlet />

// Inside components
const { id } = useParams();
const navigate = useNavigate();    // navigate("/home"), navigate(-1)
const [params, setParams] = useSearchParams();
```

### Fetching data

```jsx
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
  async function load() {
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error(`Status ${res.status}`);
      setData(await res.json());
    } catch (e) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  }
  load();
}, [url]);

if (loading) return <p>Loading...</p>;
if (error) return <p>{error}</p>;
```

### Performance

```jsx
const Page = lazy(() => import("./pages/Page"));

<Suspense fallback={<p>Loading...</p>}>
  <Page />
</Suspense>

<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>
```

---

## 11. Putting Your Project on GitHub

Your project already includes a `.gitignore` file (created by Vite) that keeps the huge `node_modules` folder out of Git.

### Step 11.1: Create an empty repository on GitHub

1. Sign in at [github.com](https://github.com).
2. Click **+** (top right), then **New repository**.
3. Name it `user-directory`. Leave "Add README" **unchecked**. Click **Create repository**.

### Step 11.2: Push your code

In the terminal, inside your `user-directory` folder:

```bash
git init
git add .
git commit -m "Build user directory app with routing, API calls and lazy loading"
git branch -M main
git remote add origin https://github.com/<your-username>/user-directory.git
git push -u origin main
```

Replace `<your-username>` with your real GitHub username. Refresh the repo page and you will see your files.

### Step 11.3: Add this guide as your README

Rename this file to `README.md` and place it in the project root folder (next to `package.json`), then:

```bash
git add README.md
git commit -m "Add beginner guide as README"
git push
```

GitHub will display it on the repository's front page.

### Step 11.4: How someone else runs your project

```bash
git clone https://github.com/<your-username>/user-directory.git
cd user-directory
npm install
npm run dev
```

### Step 11.5: Creating the production build (optional)

```bash
npm run build      # creates an optimized "dist" folder
npm run preview    # serves it locally so you can test it
```

> [!NOTE]
> Putting code on GitHub is not the same as **hosting** the app. To make it a live website, deploy the `dist` folder with a host such as Netlify, Vercel, or GitHub Pages. Because we use `BrowserRouter`, the host must be set to send `index.html` for all paths (Netlify and Vercel can do this with a simple setting).

---

### You made it!

You now know how to:

- Build multi-page React apps with **React Router**
- Load data from an **API** with proper loading and error handling
- Make apps faster with **lazy loading** and safer with **error boundaries**
- Explain **CSR vs SSR**

**Suggested next steps:** combine this app with Context or Redux Toolkit (from the State Management topic), try TanStack Query for data fetching, and explore Next.js for server-side rendering.
