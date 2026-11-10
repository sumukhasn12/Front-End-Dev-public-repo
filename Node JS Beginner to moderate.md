# Node.js — Complete Study Guide

A beginner-to-intermediate Node.js reference, organized like the GeeksforGeeks Node.js topic hub: what it is, how it works, modules, npm, building servers, REST APIs, testing, and career/ecosystem notes. Every section has runnable code.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [How Node.js Works](#2-how-nodejs-works)
3. [Installation & First Program](#3-installation--first-program)
4. [Modules](#4-modules)
   - [Core Modules](#41-core-modules)
   - [Custom Modules](#42-custom-modules)
   - [Third-Party Modules](#43-third-party-modules)
5. [npm & package.json](#5-npm--packagejson)
6. [File System (fs)](#6-file-system-fs-module)
7. [Events & EventEmitter](#7-events--eventemitter)
8. [Streams & Buffers](#8-streams--buffers)
9. [Handling Requests & Responses (http)](#9-handling-requests--responses-http)
10. [RESTful APIs](#10-restful-apis)
11. [Express.js](#11-expressjs)
12. [Async Programming](#12-async-programming)
13. [Environment Variables & Config](#13-environment-variables--config)
14. [Testing with Jest](#14-testing-with-jest)
15. [API Testing Tools: Postman vs Thunder Client](#15-api-testing-tools-postman-vs-thunder-client)
16. [Node.js vs Django vs Spring](#16-nodejs-vs-django-vs-spring)
17. [Companies, Careers & Roadmap](#17-companies-careers--roadmap)
18. [Interview Questions](#18-interview-questions)
19. [Project Structure & Best Practices](#19-project-structure--best-practices)
20. [Resources](#20-resources)

---

## 1. Introduction

**Node.js** is an open-source, cross-platform JavaScript runtime built on Chrome's **V8 engine**. It lets you run JavaScript *outside the browser*, mainly on the server.

**Key features**

| Feature | Meaning |
|---|---|
| Asynchronous & event-driven | APIs never block; callbacks/promises handle results |
| Single-threaded event loop | One main thread handles many connections efficiently |
| Fast | V8 compiles JS to machine code |
| Non-blocking I/O | Ideal for network and file-heavy apps |
| Cross-platform | Windows, macOS, Linux |
| Huge ecosystem | npm is the largest package registry |
| Same language everywhere | JavaScript on both frontend and backend |

**Good for:** REST APIs, real-time apps (chat, live dashboards), streaming, microservices, CLI tools, serverless functions.

**Not ideal for:** CPU-heavy work (video encoding, heavy number crunching) on the main thread. Use worker threads or another service.

---

## 2. How Node.js Works

Traditional servers (e.g. classic PHP/Apache) spawn a thread per request. Node uses **one thread plus an event loop**.

```
        Incoming requests
              │
              ▼
      ┌───────────────┐
      │  Event Queue  │
      └───────┬───────┘
              ▼
      ┌───────────────┐      non-blocking I/O
      │  Event Loop   │ ───────────────────────►  ┌────────────────────┐
      │ (single thread)│                          │ libuv Thread Pool   │
      └───────┬───────┘ ◄───────────────────────  │ (fs, dns, crypto…)  │
              │          callbacks/results        └────────────────────┘
              ▼
        Response sent
```

**Components**

- **V8**: executes JavaScript.
- **libuv**: C library providing the event loop, thread pool, and async I/O.
- **Node APIs (bindings)**: `fs`, `http`, `crypto`, etc.

**Event loop phases (simplified)**

1. **Timers**: `setTimeout`, `setInterval` callbacks
2. **Pending callbacks**: deferred system callbacks
3. **Poll**: retrieve new I/O events, run I/O callbacks
4. **Check**: `setImmediate` callbacks
5. **Close callbacks**: e.g. `socket.on('close')`

Between phases, `process.nextTick()` and Promise microtasks run first.

```js
console.log('1 start');
setTimeout(() => console.log('4 timeout'), 0);
setImmediate(() => console.log('5 immediate'));
Promise.resolve().then(() => console.log('3 promise'));
process.nextTick(() => console.log('2 nextTick'));
console.log('1b end');
// Order: 1 start, 1b end, 2 nextTick, 3 promise, 4 timeout, 5 immediate
// (timeout vs immediate order can vary in the main module)
```

---

## 3. Installation & First Program

### Install

- Download the **LTS** version from [nodejs.org](https://nodejs.org), or use a version manager:

```bash
# nvm (macOS/Linux)
nvm install --lts
nvm use --lts

# verify
node -v
npm -v
```

### Hello World

```js
// hello.js
console.log('Hello, Node.js!');
```

```bash
node hello.js
```

### REPL

```bash
$ node
> 2 + 3
5
> .exit
```

### Minimal web server

```js
// server.js
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from Node.js server');
});

server.listen(3000, () => console.log('Running at http://localhost:3000'));
```

---

## 4. Modules

A **module** is a file (or package) that encapsulates code and exposes only what you export. Node supports two systems:

| | CommonJS (CJS) | ES Modules (ESM) |
|---|---|---|
| Import | `const x = require('x')` | `import x from 'x'` |
| Export | `module.exports = ...` | `export default ...` / `export const ...` |
| Default for | `.js` files (no `"type"` set) | `.mjs`, or `"type": "module"` in package.json |
| Loading | Synchronous | Asynchronous, supports top-level `await` |

### 4.1 Core Modules

Built into Node. No install needed.

| Module | Purpose |
|---|---|
| `fs` | File system operations |
| `http` / `https` | Create servers and make requests |
| `path` | Work with file/directory paths |
| `os` | Operating system info |
| `events` | EventEmitter |
| `stream` | Streaming data |
| `crypto` | Hashing, encryption, random bytes |
| `url` | URL parsing |
| `util` | Utilities (`promisify`, `inspect`) |
| `child_process` | Spawn other processes |
| `worker_threads` | Parallel JS threads |
| `zlib` | Compression |
| `readline` | Read input line by line |
| `buffer` | Binary data |

```js
const path = require('path');
const os = require('os');

console.log(path.join(__dirname, 'files', 'a.txt'));
console.log(path.extname('index.html'));   // .html
console.log(os.platform(), os.cpus().length);
```

### 4.2 Custom Modules

```js
// math.js
function add(a, b) { return a + b; }
function multiply(a, b) { return a * b; }

module.exports = { add, multiply };
```

```js
// app.js
const { add, multiply } = require('./math');
console.log(add(2, 3), multiply(2, 3));
```

**ESM version**

```js
// math.mjs
export const add = (a, b) => a + b;
export default function multiply(a, b) { return a * b; }

// app.mjs
import multiply, { add } from './math.mjs';
```

**Module caching:** a module is executed once and then cached. Subsequent `require` calls return the same object.

### 4.3 Third-Party Modules

Community packages installed from npm.

```bash
npm install express
npm install lodash axios dotenv
```

```js
const _ = require('lodash');
console.log(_.chunk([1, 2, 3, 4, 5], 2)); // [[1,2],[3,4],[5]]
```

**Popular packages**

| Category | Packages |
|---|---|
| Web frameworks | express, fastify, koa, nestjs |
| Database | mongoose, sequelize, prisma, pg, mysql2 |
| Auth | jsonwebtoken, passport, bcrypt |
| Validation | joi, zod |
| HTTP client | axios, node-fetch |
| Real-time | socket.io, ws |
| Utilities | lodash, dayjs, uuid |
| Dev tools | nodemon, eslint, prettier |
| Testing | jest, mocha, supertest |

---

## 5. npm & package.json

**npm** (Node Package Manager) installs and manages dependencies. Alternatives: `yarn`, `pnpm`.

### Initialize a project

```bash
npm init -y
```

### package.json example

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.19.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0",
    "jest": "^29.0.0"
  }
}
```

### Common commands

| Command | What it does |
|---|---|
| `npm install` | Install all dependencies |
| `npm install pkg` | Install and add to `dependencies` |
| `npm install -D pkg` | Install as dev dependency |
| `npm install -g pkg` | Install globally |
| `npm uninstall pkg` | Remove package |
| `npm update` | Update packages within semver range |
| `npm outdated` | List outdated packages |
| `npm run <script>` | Run a script from package.json |
| `npm start` / `npm test` | Shortcuts for common scripts |
| `npm audit` / `npm audit fix` | Check/fix vulnerabilities |
| `npx <pkg>` | Run a package without installing globally |

### Dev dependencies

Tools needed only during development (testing, linting, hot reload) go in `devDependencies`:

```bash
npm install --save-dev nodemon jest eslint
```

In production, install only runtime deps:

```bash
npm install --omit=dev
```

### Semantic versioning

`^4.19.0` allows minor and patch updates (4.x.x). `~4.19.0` allows patch only (4.19.x). `4.19.0` pins exactly.

### Troubleshooting `npm start`

| Problem | Fix |
|---|---|
| `Missing script: "start"` | Add `"start": "node index.js"` to `scripts`, or run `node index.js` directly |
| `Cannot find module` | Run `npm install`; check the file path and spelling |
| `EADDRINUSE` | Port already in use; change port or stop the other process |
| Corrupt install | Delete `node_modules` and `package-lock.json`, then `npm install` |
| Permission errors on global installs | Use nvm or configure an npm prefix; avoid `sudo npm` |

### Running a process in the background

```bash
# Linux/macOS quick way
npm start &

# Detached and logged
nohup npm start > output.log 2>&1 &

# Recommended for real apps: a process manager
npm install -g pm2
pm2 start index.js --name my-app
pm2 logs
pm2 stop my-app
```

---

## 6. File System (fs) Module

Three styles: callbacks, sync, and promises. Prefer **promises** in modern code.

```js
const fs = require('fs');
const fsp = require('fs/promises');

// Async with promises
async function demo() {
  await fsp.writeFile('note.txt', 'Hello');
  await fsp.appendFile('note.txt', '\nWorld');
  const data = await fsp.readFile('note.txt', 'utf8');
  console.log(data);
  await fsp.rename('note.txt', 'renamed.txt');
  await fsp.unlink('renamed.txt');
}
demo().catch(console.error);

// Callback style
fs.readFile('a.txt', 'utf8', (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});

// Sync (blocks! use only at startup/scripts)
const text = fs.readFileSync('a.txt', 'utf8');
```

**Directories**

```js
await fsp.mkdir('logs/2025', { recursive: true });
const files = await fsp.readdir('logs');
await fsp.rm('logs', { recursive: true, force: true });
```

**Other useful methods:** `fs.existsSync`, `fsp.stat`, `fs.watch`, `fs.createReadStream`, `fs.createWriteStream`.

---

## 7. Events & EventEmitter

Many Node objects (servers, streams) emit events. You can build your own.

```js
const EventEmitter = require('events');

class Order extends EventEmitter {}
const order = new Order();

order.on('placed', (id) => console.log('Order placed:', id));
order.once('shipped', (id) => console.log('Shipped once:', id));

order.emit('placed', 101);
order.emit('shipped', 101);
order.emit('shipped', 101); // ignored: `once`
```

**Tips**

- Always handle the special `'error'` event, or an unhandled one will crash the process.
- Use `emitter.off(event, fn)` to remove listeners and avoid memory leaks.

---

## 8. Streams & Buffers

**Buffer**: fixed-size chunk of raw binary data.

```js
const buf = Buffer.from('Node', 'utf8');
console.log(buf);              // <Buffer 4e 6f 64 65>
console.log(buf.toString());   // Node
```

**Streams** process data piece by piece, so huge files never need to fit in memory.

| Type | Example |
|---|---|
| Readable | `fs.createReadStream`, HTTP request |
| Writable | `fs.createWriteStream`, HTTP response |
| Duplex | TCP socket |
| Transform | `zlib.createGzip()` |

```js
const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream/promises');

// Copy and compress a large file with constant memory
await pipeline(
  fs.createReadStream('big.log'),
  zlib.createGzip(),
  fs.createWriteStream('big.log.gz')
);
```

Use `pipeline` instead of `.pipe()`: it handles errors and cleanup properly.

---

## 9. Handling Requests & Responses (http)

### The request (`req`) and response (`res`)

| Object | Useful members |
|---|---|
| `req.method` | GET, POST, PUT, DELETE… |
| `req.url` | Path + query string |
| `req.headers` | Header object (lowercased keys) |
| `res.writeHead(status, headers)` | Set status and headers |
| `res.write(chunk)` / `res.end(data)` | Send body |
| `res.setHeader(name, value)` | Set a header |

### Routing and reading a JSON body by hand

```js
const http = require('http');

const users = [{ id: 1, name: 'Asha' }];

const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);

  res.setHeader('Content-Type', 'application/json');

  if (req.method === 'GET' && url.pathname === '/users') {
    res.writeHead(200);
    return res.end(JSON.stringify(users));
  }

  if (req.method === 'POST' && url.pathname === '/users') {
    let body = '';
    for await (const chunk of req) body += chunk;

    try {
      const user = { id: users.length + 1, ...JSON.parse(body) };
      users.push(user);
      res.writeHead(201);
      return res.end(JSON.stringify(user));
    } catch {
      res.writeHead(400);
      return res.end(JSON.stringify({ error: 'Invalid JSON' }));
    }
  }

  res.writeHead(404);
  res.end(JSON.stringify({ error: 'Not found' }));
});

server.listen(3000);
```

### Common HTTP status codes

| Code | Meaning |
|---|---|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 301 / 302 | Redirect |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 409 | Conflict |
| 422 | Unprocessable Entity |
| 429 | Too Many Requests |
| 500 | Internal Server Error |

### Making outgoing requests (Node 18+ has global `fetch`)

```js
const res = await fetch('https://jsonplaceholder.typicode.com/todos/1');
const data = await res.json();
console.log(data);
```

---

## 10. RESTful APIs

**REST** (Representational State Transfer) is an architectural style for networked APIs using standard HTTP.

**Principles**

1. **Client–server** separation
2. **Stateless**: each request carries everything needed
3. **Cacheable** responses
4. **Uniform interface**: resources identified by URLs
5. **Layered system**
6. (Optional) code on demand

**Resource design**

| Operation | Method | Endpoint | Success |
|---|---|---|---|
| List | GET | `/api/users` | 200 |
| Read one | GET | `/api/users/:id` | 200 |
| Create | POST | `/api/users` | 201 |
| Replace | PUT | `/api/users/:id` | 200 |
| Partial update | PATCH | `/api/users/:id` | 200 |
| Delete | DELETE | `/api/users/:id` | 204 |

**Guidelines**

- Use **nouns, plural**: `/users`, not `/getUsers`.
- Version your API: `/api/v1/...`.
- Support filtering/pagination: `/users?role=admin&page=2&limit=20`.
- Return consistent JSON error shapes: `{ "error": "message" }`.
- GET must be safe (no side effects); PUT and DELETE should be idempotent.

---

## 11. Express.js

Express is the most popular minimal web framework for Node.

```bash
npm install express
```

### Basic CRUD API

```js
const express = require('express');
const app = express();

app.use(express.json());           // parse JSON bodies

let todos = [{ id: 1, title: 'Learn Node', done: false }];

// READ all
app.get('/api/todos', (req, res) => res.json(todos));

// READ one
app.get('/api/todos/:id', (req, res) => {
  const todo = todos.find(t => t.id === Number(req.params.id));
  if (!todo) return res.status(404).json({ error: 'Not found' });
  res.json(todo);
});

// CREATE
app.post('/api/todos', (req, res) => {
  const { title } = req.body;
  if (!title) return res.status(400).json({ error: 'title is required' });
  const todo = { id: Date.now(), title, done: false };
  todos.push(todo);
  res.status(201).json(todo);
});

// UPDATE
app.put('/api/todos/:id', (req, res) => {
  const todo = todos.find(t => t.id === Number(req.params.id));
  if (!todo) return res.status(404).json({ error: 'Not found' });
  Object.assign(todo, req.body);
  res.json(todo);
});

// DELETE
app.delete('/api/todos/:id', (req, res) => {
  todos = todos.filter(t => t.id !== Number(req.params.id));
  res.sendStatus(204);
});

// 404 + error handler (must be last)
app.use((req, res) => res.status(404).json({ error: 'Route not found' }));
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: 'Something went wrong' });
});

app.listen(3000, () => console.log('API on http://localhost:3000'));
```

### Middleware

Functions that run between request and response: `(req, res, next)`.

```js
// custom logger
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});
```

Common middleware: `express.json()`, `cors`, `helmet` (security headers), `morgan` (logging), `express-rate-limit`, `cookie-parser`.

### Routers

```js
// routes/users.js
const router = require('express').Router();
router.get('/', (req, res) => res.json([]));
module.exports = router;

// app.js
app.use('/api/users', require('./routes/users'));
```

---

## 12. Async Programming

Evolution: **callbacks → promises → async/await**.

```js
// 1. Callback (can lead to "callback hell")
fs.readFile('a.txt', 'utf8', (err, a) => {
  if (err) return handle(err);
  fs.readFile('b.txt', 'utf8', (err, b) => { /* ... */ });
});

// 2. Promise
fsp.readFile('a.txt', 'utf8')
  .then(a => fsp.readFile('b.txt', 'utf8'))
  .catch(console.error);

// 3. async/await
async function main() {
  try {
    const a = await fsp.readFile('a.txt', 'utf8');
    const b = await fsp.readFile('b.txt', 'utf8');
    console.log(a + b);
  } catch (err) {
    console.error(err);
  }
}
```

**Run in parallel**

```js
const [a, b] = await Promise.all([
  fsp.readFile('a.txt', 'utf8'),
  fsp.readFile('b.txt', 'utf8'),
]);

// Promise.allSettled → never rejects, reports each outcome
// Promise.race       → first to settle wins
// Promise.any        → first to fulfill wins
```

**Convert callback APIs**

```js
const { promisify } = require('util');
const readFileAsync = promisify(fs.readFile);
```

**Global safety nets**

```js
process.on('unhandledRejection', (reason) => { console.error(reason); process.exit(1); });
process.on('uncaughtException',  (err)    => { console.error(err);    process.exit(1); });
```

**CPU-heavy work:** use `worker_threads` or `cluster` so the event loop stays responsive.

---

## 13. Environment Variables & Config

Never hard-code secrets. Use environment variables.

```bash
npm install dotenv
```

```ini
# .env  (add to .gitignore!)
PORT=3000
DB_URL=mongodb://localhost:27017/app
JWT_SECRET=change-me
```

```js
require('dotenv').config();
const port = process.env.PORT || 3000;
```

Node 20.6+ can load env files natively: `node --env-file=.env index.js`.

**.gitignore essentials**

```
node_modules/
.env
logs/
*.log
dist/
coverage/
```

---

## 14. Testing with Jest

**Jest** is a JavaScript testing framework with a runner, assertions, mocks, and coverage built in.

```bash
npm install --save-dev jest
```

```json
"scripts": { "test": "jest" }
```

### Unit test

```js
// sum.js
module.exports = (a, b) => a + b;

// sum.test.js
const sum = require('./sum');

describe('sum', () => {
  test('adds two numbers', () => {
    expect(sum(2, 3)).toBe(5);
  });

  test('handles negatives', () => {
    expect(sum(-1, -1)).toBe(-2);
  });
});
```

### Common matchers

`toBe`, `toEqual`, `toBeTruthy`, `toBeNull`, `toContain`, `toHaveLength`, `toThrow`, `toHaveBeenCalledWith`, `resolves`, `rejects`.

### Async & mocks

```js
test('fetches user', async () => {
  const user = await getUser(1);
  expect(user.name).toBe('Asha');
});

test('calls callback', () => {
  const fn = jest.fn();
  run(fn);
  expect(fn).toHaveBeenCalledTimes(1);
});

jest.mock('./db');   // auto-mock a module
```

### API tests with Supertest

```bash
npm install --save-dev supertest
```

```js
const request = require('supertest');
const app = require('./app');   // export app without calling listen()

test('GET /api/todos', async () => {
  const res = await request(app).get('/api/todos');
  expect(res.status).toBe(200);
  expect(Array.isArray(res.body)).toBe(true);
});
```

Run with coverage: `npx jest --coverage`. Watch mode: `npx jest --watch`.

---

## 15. API Testing Tools: Postman vs Thunder Client

**Thunder Client** is a lightweight REST client that runs as a VS Code extension. **Postman** is a full-featured standalone platform.

| Feature | Postman | Thunder Client |
|---|---|---|
| Where it runs | Standalone app / web | Inside VS Code |
| Footprint | Heavier | Lightweight |
| Collections & environments | Yes | Yes |
| Automated test scripts | Powerful (JS scripting) | Basic |
| Mock servers, monitors | Yes | No / limited |
| Team collaboration | Strong (workspaces) | Limited |
| Best for | Large teams, full API lifecycle | Quick tests while coding |

**Basic workflow in either tool**

1. Choose method and enter URL, e.g. `http://localhost:3000/api/todos`.
2. Add headers, e.g. `Content-Type: application/json`.
3. For POST/PUT, put JSON in the body: `{ "title": "Test" }`.
4. Send and inspect status, time, and response body.
5. Save the request into a collection; use environment variables (`{{baseUrl}}`).

You can also use `curl`:

```bash
curl -X POST http://localhost:3000/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Test"}'
```

---

## 16. Node.js vs Django vs Spring

| Aspect | Node.js (Express) | Django | Spring (Boot) |
|---|---|---|---|
| Language | JavaScript / TypeScript | Python | Java / Kotlin |
| Type | Runtime + minimal framework | Full-stack, "batteries included" | Enterprise framework |
| Concurrency | Event loop, non-blocking | Threads/processes (async supported) | Multithreaded (reactive option) |
| Learning curve | Gentle | Gentle | Steeper |
| Performance | Great for I/O-bound | Good | Excellent, strong for CPU-bound |
| Built-ins | Minimal; pick your own libs | ORM, admin, auth, forms | DI, security, data, huge ecosystem |
| Best for | Real-time apps, APIs, microservices | Content sites, rapid MVPs, data apps | Large enterprise/banking systems |
| Typical users | Startups, streaming, fintech APIs | Instagram-style, ML-backed apps | Banks, large enterprises |

**Rule of thumb:** pick the one your team knows best, then match it to the workload. Node for I/O-heavy and real-time, Django for fast full-featured builds, Spring for large, long-lived enterprise systems.

---

## 17. Companies, Careers & Roadmap

### Companies known for using Node.js

Netflix, PayPal, LinkedIn, Uber, Walmart, NASA, Trello, eBay, Medium, GoDaddy. (Check each company's engineering blog for current usage.)

### Career paths

- Backend Developer
- Full-Stack Developer (MERN/MEAN)
- API / Microservices Engineer
- DevOps / Cloud Engineer (serverless, containers)
- Technical Lead / Architect

### Learning roadmap

```
JavaScript fundamentals (ES6+, async/await)
        ↓
Node basics: modules, fs, http, npm
        ↓
Express + REST API design
        ↓
Database: MongoDB (Mongoose) or PostgreSQL (Prisma/Sequelize)
        ↓
Auth: JWT, sessions, bcrypt, OAuth
        ↓
Testing: Jest, Supertest
        ↓
TypeScript + validation (zod/joi)
        ↓
Real-time: WebSockets / Socket.io
        ↓
Deployment: Docker, CI/CD, cloud (AWS/Azure/GCP), PM2, Nginx
        ↓
Advanced: caching (Redis), queues (BullMQ), microservices, GraphQL, NestJS
```

---

## 18. Interview Questions

**Basics**

1. What is Node.js and why is it single-threaded yet scalable?
2. Explain the event loop and its phases.
3. Difference between `process.nextTick()`, `setImmediate()`, and `setTimeout()`?
4. What is the difference between `require` and `import`?
5. What are `dependencies` vs `devDependencies`?
6. What is `package-lock.json` for?

**Intermediate**

7. Blocking vs non-blocking I/O, with an example.
8. What are streams? Why use `pipeline`?
9. Explain middleware in Express.
10. How does `module.exports` differ from `exports`?
11. What is callback hell and how do you avoid it?
12. How do you handle errors in async code and in Express?
13. What is the `cluster` module and when do you use it?
14. How do you secure a Node app (helmet, rate limiting, input validation, CORS, env secrets)?

**Advanced**

15. How do worker threads differ from the cluster module?
16. How would you scale a Node app horizontally?
17. How do you find and fix memory leaks?
18. Explain backpressure in streams.
19. What is graceful shutdown and how do you implement it?
20. REST vs GraphQL: trade-offs?

**Quick answers**

- `nextTick` runs before promise microtasks and before the event loop continues; `setImmediate` runs in the check phase; `setTimeout(fn, 0)` runs in the timers phase.
- `exports` is a reference to `module.exports`. Reassigning `exports = ...` breaks the link; assign to `module.exports` instead.
- `package-lock.json` locks exact dependency versions so installs are reproducible.

---

## 19. Project Structure & Best Practices

### Suggested structure

```
my-app/
├── src/
│   ├── config/          # env, db connection
│   ├── controllers/     # request handlers
│   ├── routes/          # route definitions
│   ├── services/        # business logic
│   ├── models/          # DB schemas
│   ├── middleware/      # auth, validation, error handling
│   ├── utils/
│   ├── app.js           # express app (exported, no listen)
│   └── server.js        # starts the server
├── tests/
├── .env
├── .gitignore
├── package.json
└── README.md
```

### Best practices checklist

- [ ] Use environment variables for secrets and config
- [ ] Validate all input (zod/joi)
- [ ] Centralize error handling
- [ ] Prefer async/await; never block the event loop with sync calls
- [ ] Use `helmet`, `cors`, and rate limiting
- [ ] Hash passwords with bcrypt/argon2
- [ ] Log with a real logger (pino/winston)
- [ ] Lint and format (ESLint, Prettier)
- [ ] Write tests and run them in CI
- [ ] Keep dependencies updated; run `npm audit`
- [ ] Handle `SIGTERM` for graceful shutdown
- [ ] Use a process manager or containers in production

### Graceful shutdown

```js
const server = app.listen(3000);

process.on('SIGTERM', () => {
  server.close(() => {
    // close DB connections here
    process.exit(0);
  });
});
```

---

## 20. Resources

- Official docs: https://nodejs.org/docs
- npm registry: https://www.npmjs.com
- Express: https://expressjs.com
- Jest: https://jestjs.io
- MDN JavaScript guide: https://developer.mozilla.org/docs/Web/JavaScript
- GeeksforGeeks Node.js hub: https://www.geeksforgeeks.org/node-js
- Node.js Best Practices (community repo): https://github.com/goldbergyoni/nodebestpractices

---

## Contributing

Pull requests are welcome. For major changes, open an issue first to discuss what you'd like to change.

## License

MIT
