# Node.js — Intermediate to Advanced Guide

For students who already know Node.js basics (modules, `fs`, `http`, Express, npm, async/await). This guide goes into **how Node really works internally**, **performance**, **scalability**, **security**, **production architecture**, and **hard interview/practice problems**.

**Prerequisites:** solid JavaScript (closures, promises, classes), basic Express, basic SQL or MongoDB, comfort with the terminal and Git.

**Difficulty tags used below:** 🟡 Moderate · 🔴 Difficult · ⚫ Expert

---

## Table of Contents

1. [Event Loop Deep Dive](#1-event-loop-deep-dive) 🟡🔴
2. [Async Patterns & Concurrency Control](#2-async-patterns--concurrency-control) 🟡
3. [Streams, Backpressure & Pipelines](#3-streams-backpressure--pipelines) 🔴
4. [Worker Threads, Cluster & Child Processes](#4-worker-threads-cluster--child-processes) 🔴
5. [Error Handling Strategy](#5-error-handling-strategy) 🟡
6. [Performance & Profiling](#6-performance--profiling) 🔴
7. [Memory Management & Leaks](#7-memory-management--leaks) 🔴⚫
8. [AsyncLocalStorage & Request Context](#8-asynclocalstorage--request-context) 🔴
9. [Authentication & Authorization](#9-authentication--authorization) 🟡🔴
10. [Security Hardening](#10-security-hardening) 🔴
11. [Databases: Pooling, Transactions, Indexing](#11-databases-pooling-transactions-indexing) 🔴
12. [Caching with Redis](#12-caching-with-redis) 🔴
13. [Queues & Background Jobs](#13-queues--background-jobs) 🔴
14. [Rate Limiting & Resilience Patterns](#14-rate-limiting--resilience-patterns) 🔴
15. [Real-Time: WebSockets](#15-real-time-websockets) 🟡🔴
16. [Architecture & Design Patterns](#16-architecture--design-patterns) 🔴
17. [Advanced Testing](#17-advanced-testing) 🟡🔴
18. [TypeScript with Node](#18-typescript-with-node) 🟡
19. [ESM, CJS Interop & Module Internals](#19-esm-cjs-interop--module-internals) 🔴
20. [Docker, CI/CD & Production Deployment](#20-docker-cicd--production-deployment) 🟡🔴
21. [Observability: Logs, Metrics, Tracing](#21-observability-logs-metrics-tracing) 🔴
22. [Microservices & Messaging](#22-microservices--messaging) ⚫
23. [Hard Interview Questions](#23-hard-interview-questions)
24. [Practice Problems](#24-practice-problems)
25. [Capstone Projects](#25-capstone-projects)
26. [Resources](#26-resources)

---

## 1. Event Loop Deep Dive

### 1.1 The architecture

```
 JavaScript (V8)  ──►  Node bindings (C++)  ──►  libuv
   call stack            fs, net, crypto         event loop + thread pool
```

- **Network I/O** (sockets) uses the OS's async mechanism (epoll/kqueue/IOCP). No thread-pool thread is used.
- **File system, DNS (`dns.lookup`), some crypto (`pbkdf2`, `scrypt`, `randomBytes`), zlib** use the **libuv thread pool** (default size **4**, tune with `UV_THREADPOOL_SIZE`, max 1024).

### 1.2 Phases of one loop iteration

```
   ┌─► timers            setTimeout / setInterval
   │   pending callbacks  deferred system errors (e.g. TCP ECONNREFUSED)
   │   idle, prepare      internal
   │   poll               fetch I/O events, run I/O callbacks (may block here)
   │   check              setImmediate
   └── close callbacks    socket.on('close')

 Between EVERY callback:  process.nextTick queue  →  microtask (Promise) queue
```

### 1.3 Predict the output 🔴

```js
const fs = require('fs');

setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

fs.readFile(__filename, () => {
  setTimeout(() => console.log('fs → timeout'), 0);
  setImmediate(() => console.log('fs → immediate'));
  process.nextTick(() => console.log('fs → nextTick'));
});

Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
```

**Answer**

```
nextTick
promise
timeout / immediate     ← order NOT guaranteed from the main module
fs → nextTick
fs → immediate          ← inside an I/O callback, immediate ALWAYS beats timeout
fs → timeout
```

**Why:** from the main module, whether the 1 ms timer has expired when the loop starts depends on process performance. Inside an I/O callback you are in the poll phase, so the next phase is *check* (immediate) before looping back to *timers*.

### 1.4 `nextTick` starvation 🔴

```js
function recurse() { process.nextTick(recurse); }
recurse();                     // I/O callbacks and timers NEVER run
setTimeout(() => console.log('never printed'), 0);
```

`nextTick` and microtask queues drain completely before the loop continues. Recursive scheduling starves I/O. Use `setImmediate` to yield to the loop.

### 1.5 Blocking the loop

```js
// BAD: blocks ALL requests for the duration
app.get('/fib', (req, res) => res.json({ n: fibSync(45) }));

// Measure event loop delay
const { monitorEventLoopDelay } = require('perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  console.log('p99 delay ms:', h.percentile(99) / 1e6);
  h.reset();
}, 5000);
```

**Common loop blockers:** `JSON.parse/stringify` of huge payloads, sync `fs` calls, catastrophic regex (ReDoS), big loops, `crypto.pbkdf2Sync`, large array sorts.

### 1.6 Partitioning long work

```js
async function processBig(items) {
  for (let i = 0; i < items.length; i++) {
    handle(items[i]);
    if (i % 1000 === 0) await new Promise(setImmediate); // yield to loop
  }
}
```

---

## 2. Async Patterns & Concurrency Control

### 2.1 Sequential vs parallel vs limited 🟡

```js
// Sequential (slow if independent)
for (const id of ids) await fetchUser(id);

// Unbounded parallel (dangerous for 10,000 ids: socket/DB exhaustion)
await Promise.all(ids.map(fetchUser));

// Bounded concurrency: implement yourself
async function mapLimit(items, limit, fn) {
  const results = new Array(items.length);
  let next = 0;

  async function worker() {
    while (true) {
      const i = next++;
      if (i >= items.length) return;
      results[i] = await fn(items[i], i);
    }
  }

  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}

const users = await mapLimit(ids, 5, fetchUser);
```

(Production alternative: `p-limit`, `p-map`.)

### 2.2 Promise combinators

| Method | Resolves when | Rejects when |
|---|---|---|
| `Promise.all` | all fulfil | any rejects (fail-fast) |
| `Promise.allSettled` | all settle | never |
| `Promise.race` | first settles | first settles as rejected |
| `Promise.any` | first fulfils | all reject (`AggregateError`) |

### 2.3 Timeouts and cancellation with `AbortController` 🟡

```js
async function fetchWithTimeout(url, ms) {
  const ac = new AbortController();
  const t = setTimeout(() => ac.abort(), ms);
  try {
    return await fetch(url, { signal: ac.signal });
  } finally {
    clearTimeout(t);
  }
}

// Built-in helper (Node 17.3+)
await fetch(url, { signal: AbortSignal.timeout(3000) });
```

### 2.4 Retry with exponential backoff and jitter 🔴

```js
async function retry(fn, { retries = 5, base = 200, max = 5000 } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn(attempt);
    } catch (err) {
      if (attempt >= retries || !isRetryable(err)) throw err;
      const exp = Math.min(max, base * 2 ** attempt);
      const delay = Math.random() * exp;                // full jitter
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

const isRetryable = (e) => ['ECONNRESET', 'ETIMEDOUT', 'EAI_AGAIN'].includes(e.code)
  || (e.status >= 500 && e.status < 600);
```

### 2.5 Pitfalls

```js
// 1. forEach does NOT await
items.forEach(async (x) => { await save(x); });  // fire-and-forget!

// 2. Forgotten return inside then-chain
p.then(() => { doAsync(); })       // not awaited
 .then(() => console.log('too early'));

// 3. Unhandled rejections crash modern Node (v15+)
async function bad() { throw new Error('x'); }
bad();  // process exits with unhandledRejection

// 4. Mixed callback + promise returning twice
```

---

## 3. Streams, Backpressure & Pipelines

### 3.1 Why backpressure matters 🔴

If a producer writes faster than a consumer can handle, data buffers in memory → OOM. A writable signals this via `write()` returning `false` and later emitting `'drain'`.

```js
// Manual handling (to understand what pipeline does for you)
function copy(src, dest) {
  src.on('data', (chunk) => {
    if (!dest.write(chunk)) {
      src.pause();
      dest.once('drain', () => src.resume());
    }
  });
  src.on('end', () => dest.end());
}
```

### 3.2 Always use `pipeline`

```js
const { pipeline } = require('stream/promises');
const { Transform } = require('stream');
const fs = require('fs');
const zlib = require('zlib');

const upper = new Transform({
  transform(chunk, enc, cb) { cb(null, chunk.toString().toUpperCase()); },
});

await pipeline(
  fs.createReadStream('in.txt'),
  upper,
  zlib.createGzip(),
  fs.createWriteStream('out.txt.gz'),
);
```

`pipeline` destroys all streams on error and propagates the error; `.pipe()` does not.

### 3.3 Object mode and async generators 🔴

```js
const { Readable } = require('stream');

async function* numbers() {
  for (let i = 1; i <= 1e6; i++) yield { id: i };
}

await pipeline(
  Readable.from(numbers()),                       // object mode by default
  async function* (source) {                      // async generator as transform
    for await (const row of source) {
      if (row.id % 2 === 0) yield JSON.stringify(row) + '\n';
    }
  },
  fs.createWriteStream('even.ndjson'),
);
```

### 3.4 Streaming a large CSV to an HTTP response

```js
app.get('/export', async (req, res, next) => {
  res.setHeader('Content-Type', 'text/csv');
  try {
    await pipeline(db.streamRows('SELECT * FROM orders'), toCsvTransform(), res);
  } catch (e) { next(e); }
});
```

### 3.5 Web Streams

Node supports WHATWG streams (`ReadableStream`, `TransformStream`), used by `fetch` response bodies. Convert with `Readable.fromWeb(webStream)` and `Readable.toWeb(nodeStream)`.

---

## 4. Worker Threads, Cluster & Child Processes

### 4.1 Choosing the tool

| Tool | What it is | Use when |
|---|---|---|
| `cluster` | Multiple **processes** sharing a server port | Scale an HTTP server across CPU cores |
| `worker_threads` | Threads in the same process, own V8 isolate | CPU-heavy JS tasks (parsing, image/crypto, math) |
| `child_process` | Spawn external programs/processes | Run shell commands, other languages |
| Containers/PM2 | Orchestrated process replicas | Production scaling (preferred over `cluster` code) |

### 4.2 Worker threads 🔴

```js
// main.js
const { Worker } = require('worker_threads');

function runWorker(data) {
  return new Promise((resolve, reject) => {
    const w = new Worker('./worker.js', { workerData: data });
    w.once('message', resolve);
    w.once('error', reject);
    w.once('exit', (code) => code !== 0 && reject(new Error(`exit ${code}`)));
  });
}

app.get('/fib/:n', async (req, res) => {
  res.json({ result: await runWorker(Number(req.params.n)) });
});
```

```js
// worker.js
const { parentPort, workerData } = require('worker_threads');
const fib = (n) => (n < 2 ? n : fib(n - 1) + fib(n - 2));
parentPort.postMessage(fib(workerData));
```

**Notes**

- Spawning a worker per request is expensive. Use a **worker pool** (`piscina` library, or build your own).
- Data is copied via structured clone. For big binary data use **transferables** (`ArrayBuffer`) or `SharedArrayBuffer` + `Atomics`.

```js
// Shared memory counter
const sab = new SharedArrayBuffer(4);
const counter = new Int32Array(sab);
Atomics.add(counter, 0, 1);   // thread-safe increment
```

### 4.3 Cluster 🟡

```js
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork();
  cluster.on('exit', (worker, code) => {
    console.log(`worker ${worker.process.pid} died (${code}); restarting`);
    cluster.fork();
  });
} else {
  require('./server');   // each worker listens on the same port
}
```

Because workers are separate processes, **in-memory state is not shared**. Use Redis or a DB for sessions, rate-limit counters and caches.

### 4.4 Child processes

```js
const { execFile, spawn } = require('child_process');
const { promisify } = require('util');

// Short output, buffered: prefer execFile (no shell → safer than exec)
const { stdout } = await promisify(execFile)('git', ['rev-parse', 'HEAD']);

// Long-running / streaming output
const p = spawn('ffmpeg', ['-i', 'in.mp4', 'out.webm']);
p.stderr.on('data', (d) => process.stderr.write(d));
p.on('close', (code) => console.log('exit', code));
```

**Security:** never pass user input to `exec()`; it uses a shell → command injection.

---

## 5. Error Handling Strategy

### 5.1 Operational vs programmer errors 🟡

| | Operational | Programmer |
|---|---|---|
| Examples | DB down, invalid input, timeout | `undefined.foo`, wrong arg type |
| Handle by | Retry, respond with 4xx/5xx, log | Fix the bug; **crash and restart** |

### 5.2 Custom error hierarchy

```js
class AppError extends Error {
  constructor(message, { status = 500, code = 'INTERNAL', cause, details } = {}) {
    super(message, { cause });
    this.name = this.constructor.name;
    this.status = status;
    this.code = code;
    this.details = details;
    this.isOperational = true;
    Error.captureStackTrace?.(this, this.constructor);
  }
}
class NotFoundError extends AppError {
  constructor(what) { super(`${what} not found`, { status: 404, code: 'NOT_FOUND' }); }
}
class ValidationError extends AppError {
  constructor(details) { super('Invalid input', { status: 422, code: 'VALIDATION', details }); }
}
```

### 5.3 Express async wrapper + central handler

```js
const wrap = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);

app.get('/users/:id', wrap(async (req, res) => {
  const u = await users.find(req.params.id);
  if (!u) throw new NotFoundError('User');
  res.json(u);
}));

app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);
  const status = err.status ?? 500;
  if (status >= 500) req.log.error({ err }, 'unhandled');
  res.status(status).json({
    error: { code: err.code ?? 'INTERNAL', message: status < 500 ? err.message : 'Internal error' },
  });
});
```

(Express 5 forwards rejected promises from async handlers automatically.)

### 5.4 Process-level handlers

```js
process.on('unhandledRejection', (reason) => { logger.fatal({ reason }); throw reason; });
process.on('uncaughtException', (err) => {
  logger.fatal({ err });
  shutdown(1);   // state may be corrupt: exit and let the supervisor restart
});
```

### 5.5 Error `cause`

```js
try { await db.query(sql); }
catch (e) { throw new AppError('Could not load orders', { cause: e }); }
```

---

## 6. Performance & Profiling

### 6.1 Measure first

| Tool | Purpose |
|---|---|
| `autocannon` / `wrk` / `k6` | Load testing |
| `node --prof` + `--prof-process` | V8 CPU profile |
| `node --inspect` + Chrome DevTools | CPU/heap profiling, breakpoints |
| `clinic doctor / flame / bubbleprof` | Diagnose event-loop, CPU, async issues |
| `0x` | Flamegraphs |
| `perf_hooks` | Custom timing and event-loop delay |

```bash
npx autocannon -c 100 -d 20 http://localhost:3000/api/items
node --inspect server.js     # then open chrome://inspect
```

### 6.2 Timing code

```js
const { performance, PerformanceObserver } = require('perf_hooks');

performance.mark('start');
await work();
performance.mark('end');
performance.measure('work', 'start', 'end');
console.log(performance.getEntriesByName('work')[0].duration, 'ms');
```

### 6.3 Practical optimizations 🔴

| Problem | Fix |
|---|---|
| Slow JSON serialization | Schema-based serializers (`fast-json-stringify`, Fastify) |
| Repeated DB reads | Cache (Redis/in-memory LRU), batch with DataLoader |
| N+1 queries | Joins, `IN (...)` batching, ORM eager loading |
| Large payloads | Compression (`compression`), pagination, streaming |
| Sync CPU work | Worker threads, offload to a queue |
| Too many sockets | Reuse connections with `http.Agent({ keepAlive: true })` |
| Cold regex | Precompile regex outside hot paths; avoid catastrophic patterns |
| Thread pool saturation | Raise `UV_THREADPOOL_SIZE`; avoid sync crypto |

```js
const http = require('http');
const agent = new http.Agent({ keepAlive: true, maxSockets: 50 });
```

### 6.4 V8 notes ⚫

- Keep object **shapes (hidden classes)** stable: initialize all properties in the constructor in the same order.
- Avoid changing a variable's type in hot code (monomorphic vs polymorphic call sites).
- Avoid `delete obj.prop` in hot paths; set to `undefined` or use `Map`.

---

## 7. Memory Management & Leaks

### 7.1 Basics

- V8 heap: **young generation** (short-lived, fast scavenges) and **old generation** (mark-sweep-compact).
- Buffers live **outside** the V8 heap (`external` / `arrayBuffers` memory).
- Default old-space limit depends on system memory; raise with `--max-old-space-size=4096`.

```js
setInterval(() => {
  const m = process.memoryUsage();
  console.log({
    rssMB: (m.rss / 1048576).toFixed(0),
    heapUsedMB: (m.heapUsed / 1048576).toFixed(0),
    externalMB: (m.external / 1048576).toFixed(0),
  });
}, 10000);
```

### 7.2 Classic leak sources 🔴

```js
// 1. Unbounded cache / global collections
const cache = {};                              // grows forever
app.get('/u/:id', async (req, res) => { cache[req.params.id] = await load(req.params.id); });
// Fix: LRU with max size/TTL (lru-cache), or WeakMap for object-keyed data

// 2. Listeners never removed
function handler(socket) {
  emitter.on('tick', () => socket.write('x'));   // added per connection, never removed
}
// Fix: emitter.off in socket 'close'. Watch for MaxListenersExceededWarning.

// 3. Timers that outlive their owner
const t = setInterval(poll, 1000);   // keep reference; clearInterval(t) on teardown
// t.unref() lets the process exit even if timer pending

// 4. Closures capturing large scope
function make() {
  const big = Buffer.alloc(1e8);
  return () => 'small';        // may retain `big` depending on engine optimizations
}

// 5. Promises that never settle, holding their continuations
```

### 7.3 Finding leaks

```bash
node --inspect server.js
```

1. Open DevTools → Memory → take **Heap snapshot** (baseline).
2. Run load; force GC; take second snapshot.
3. Use **Comparison** view: sort by "Delta"/"Size Delta"; inspect **Retainers** to find what holds references.

Programmatic:

```js
const v8 = require('v8');
v8.writeHeapSnapshot();                // writes .heapsnapshot file
// Or run with: node --heapsnapshot-signal=SIGUSR2 server.js
```

### 7.4 WeakRef and FinalizationRegistry ⚫

```js
const cache = new Map();
function getCached(key, load) {
  const ref = cache.get(key);
  const hit = ref?.deref();
  if (hit) return hit;
  const val = load(key);
  cache.set(key, new WeakRef(val));
  return val;
}
```

Use sparingly; GC timing is non-deterministic.

---

## 8. AsyncLocalStorage & Request Context

Carry per-request data (request ID, user, tenant) through async calls **without passing it as a parameter**.

```js
const { AsyncLocalStorage } = require('node:async_hooks');
const crypto = require('crypto');

const als = new AsyncLocalStorage();

app.use((req, res, next) => {
  als.run({ reqId: crypto.randomUUID(), user: null }, next);
});

function getCtx() { return als.getStore(); }

// deep inside any service/repository function:
function log(msg) {
  console.log(JSON.stringify({ reqId: getCtx()?.reqId, msg }));
}
```

**Use cases:** correlation IDs in logs, multi-tenant DB selection, tracing, transaction propagation. Context follows `await`, promises, timers and callbacks started inside `run()`.

---

## 9. Authentication & Authorization

### 9.1 Password storage 🟡

```js
const argon2 = require('argon2');          // or bcrypt
const hash = await argon2.hash(password);
const ok   = await argon2.verify(hash, attempt);
```

Never use plain SHA/MD5 for passwords. Always salted, slow, adaptive hashing.

### 9.2 JWT access + refresh token flow 🔴

```
Login ──► access token (15 min, in memory / Authorization header)
      └─► refresh token (7–30 days, httpOnly+Secure+SameSite cookie, stored hashed in DB)

Access expired ──► POST /auth/refresh with cookie ──► NEW access + ROTATED refresh
                   Old refresh token invalidated. Reuse of an old one ⇒ revoke the whole family.
```

```js
const jwt = require('jsonwebtoken');

const signAccess = (user) =>
  jwt.sign({ sub: user.id, role: user.role }, process.env.JWT_SECRET, {
    expiresIn: '15m', algorithm: 'HS256', issuer: 'my-api', audience: 'my-web',
  });

function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.sendStatus(401);
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET, {
      algorithms: ['HS256'],           // ALWAYS pin the algorithm
      issuer: 'my-api', audience: 'my-web',
    });
    next();
  } catch { res.sendStatus(401); }
}

const authorize = (...roles) => (req, res, next) =>
  roles.includes(req.user.role) ? next() : res.sendStatus(403);

app.delete('/users/:id', authenticate, authorize('admin'), handler);
```

**JWT pitfalls**

- Pinning `algorithms` prevents `alg: none` and HS/RS confusion attacks.
- JWTs can't be revoked before expiry without a denylist. Keep access tokens short.
- Don't put sensitive data in the payload; it's only base64-encoded, not encrypted.
- Store refresh tokens as **hashes** in the DB, like passwords.

### 9.3 Sessions vs JWT

| | Server sessions | JWT |
|---|---|---|
| State | Server (Redis/DB) | Client (stateless) |
| Revocation | Easy | Hard |
| Scaling | Needs shared store | Easy |
| Best for | Traditional web apps | APIs, mobile, microservices |

### 9.4 Authorization models

- **RBAC**: roles → permissions.
- **ABAC / policy-based**: rules over attributes (`user.dept === doc.dept`).
- **Ownership checks**: always verify the resource belongs to the caller (prevents **IDOR/BOLA**).

```js
const order = await Orders.findById(req.params.id);
if (!order || order.userId !== req.user.sub) return res.sendStatus(404);
```

### 9.5 OAuth 2.0 / OIDC overview

Use the **Authorization Code flow with PKCE** for SPAs and mobile. Use a vetted library or provider (Auth0, Keycloak, Cognito, `openid-client`); don't hand-roll.

---

## 10. Security Hardening

### 10.1 OWASP-driven checklist 🔴

| Risk | Defence in Node |
|---|---|
| Injection (SQL/NoSQL/command) | Parameterized queries; never `exec(userInput)`; sanitize Mongo operators |
| Broken auth | Strong hashing, rate-limited login, MFA, short-lived tokens |
| Sensitive data exposure | HTTPS/HSTS, encrypt at rest, never log secrets |
| XSS | Escape output, CSP header, `helmet` |
| CSRF | SameSite cookies, CSRF tokens for cookie-auth |
| SSRF | Allowlist outbound hosts; block private IP ranges |
| Mass assignment | Whitelist fields (zod), never `Model.create(req.body)` |
| ReDoS | Avoid nested quantifiers; use `safe-regex`/`re2` |
| Prototype pollution | Reject `__proto__`, `constructor` keys; use `Object.create(null)`, `Map` |
| Vulnerable deps | `npm audit`, Dependabot/Renovate, lockfile, `npm ci` |
| Path traversal | Resolve and verify paths stay inside a base dir |

### 10.2 Code samples

```js
// SQL injection
await pool.query('SELECT * FROM users WHERE email = $1', [email]);   // GOOD
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);    // BAD

// NoSQL injection: body { "email": { "$ne": null } }
if (typeof req.body.email !== 'string') return res.sendStatus(400);

// Path traversal
const path = require('path');
const BASE = path.resolve('./uploads');
const target = path.resolve(BASE, req.params.file);
if (!target.startsWith(BASE + path.sep)) return res.sendStatus(400);

// Timing-safe comparison
const crypto = require('crypto');
const a = Buffer.from(sig), b = Buffer.from(expected);
const valid = a.length === b.length && crypto.timingSafeEqual(a, b);
```

### 10.3 Express hardening baseline

```js
const helmet = require('helmet');
const cors = require('cors');

app.disable('x-powered-by');
app.use(helmet());
app.use(cors({ origin: ['https://app.example.com'], credentials: true }));
app.use(express.json({ limit: '100kb' }));       // cap body size
app.set('trust proxy', 1);                       // correct IPs behind a proxy
```

### 10.4 Validation with Zod

```js
const { z } = require('zod');

const CreateUser = z.object({
  email: z.string().email(),
  password: z.string().min(12).max(128),
  age: z.number().int().min(13).optional(),
}).strict();                                       // reject unknown keys

const validate = (schema) => (req, res, next) => {
  const r = schema.safeParse(req.body);
  if (!r.success) return res.status(422).json({ errors: r.error.flatten() });
  req.body = r.data;
  next();
};
```

### 10.5 Secrets management

- Never commit `.env`; use a secrets manager (AWS Secrets Manager, Vault, Doppler).
- Rotate secrets; separate per environment.
- Run `git-secrets`/`gitleaks` in CI.
- Node's **permission model** (`--permission`, Node 20+) can restrict fs/child-process access.

---

## 11. Databases: Pooling, Transactions, Indexing

### 11.1 Connection pooling 🔴

Opening a connection per request is slow and exhausts DB limits.

```js
const { Pool } = require('pg');
const pool = new Pool({
  max: 20,                       // size relative to DB max_connections / app instances
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
});
```

**Math:** `instances × pool.max` must stay under the DB's `max_connections`. Consider PgBouncer for many instances/serverless.

### 11.2 Transactions

```js
async function transfer(from, to, amount) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows } = await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1 RETURNING balance',
      [amount, from],
    );
    if (rows.length === 0) throw new Error('Insufficient funds');
    await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to]);
    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();        // ALWAYS release, or the pool leaks
  }
}
```

**Isolation levels and anomalies**

| Level | Dirty read | Non-repeatable | Phantom |
|---|---|---|---|
| Read Uncommitted | possible | possible | possible |
| Read Committed (PG default) | no | possible | possible |
| Repeatable Read | no | no | possible* |
| Serializable | no | no | no |

*PostgreSQL's Repeatable Read also prevents phantoms; Serializable adds write-skew protection and may require retrying on serialization failures.

### 11.3 Concurrency control

```sql
-- Pessimistic lock
SELECT * FROM inventory WHERE id = $1 FOR UPDATE;

-- Optimistic locking with version column
UPDATE products SET stock = stock - 1, version = version + 1
WHERE id = $1 AND version = $2;       -- 0 rows updated ⇒ conflict, retry
```

**Idempotency keys** for payments/orders: store the key with a unique constraint so a retried request can't create a duplicate.

### 11.4 N+1 and DataLoader

```js
// N+1: 1 query for posts + N queries for authors
for (const p of posts) p.author = await Users.findById(p.authorId);

// Batched
const ids = [...new Set(posts.map(p => p.authorId))];
const users = await Users.find({ id: { $in: ids } });
const byId = new Map(users.map(u => [u.id, u]));
posts.forEach(p => (p.author = byId.get(p.authorId)));
```

### 11.5 Indexing essentials

- Index columns used in `WHERE`, `JOIN`, `ORDER BY`.
- Composite index order matters: `(a, b)` helps queries on `a` or `a+b`, not `b` alone.
- Use `EXPLAIN ANALYZE` (SQL) / `.explain('executionStats')` (MongoDB) before and after.
- Prefer **keyset pagination** over `OFFSET` for large tables:

```sql
SELECT * FROM posts WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC LIMIT 20;
```

### 11.6 Migrations

Use versioned migrations (Prisma Migrate, Knex, node-pg-migrate). Make changes **backward compatible** (expand → migrate data → contract) so you can deploy without downtime.

---

## 12. Caching with Redis

### 12.1 Cache-aside pattern 🔴

```js
const { createClient } = require('redis');
const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

async function getProduct(id) {
  const key = `product:${id}`;
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit);

  const row = await db.products.find(id);
  if (row) await redis.set(key, JSON.stringify(row), { EX: 300 });   // TTL
  return row;
}

async function updateProduct(id, data) {
  await db.products.update(id, data);
  await redis.del(`product:${id}`);          // invalidate
}
```

### 12.2 Cache problems to know

| Problem | Description | Mitigation |
|---|---|---|
| **Stampede** | Hot key expires, thousands hit the DB | Request coalescing/locking, jittered TTLs, stale-while-revalidate |
| **Penetration** | Requests for nonexistent keys bypass cache | Cache negative results briefly; Bloom filter |
| **Avalanche** | Many keys expire together | Randomize TTLs |
| **Stale data** | Cache differs from DB | Invalidate on write; short TTLs |

**Request coalescing (single-flight)**

```js
const inflight = new Map();
function singleFlight(key, loader) {
  if (inflight.has(key)) return inflight.get(key);
  const p = loader().finally(() => inflight.delete(key));
  inflight.set(key, p);
  return p;
}
```

### 12.3 Other Redis uses

- Sessions, rate limiting counters (`INCR` + `EXPIRE`)
- Distributed locks (Redlock, with caution; use fencing tokens for correctness-critical work)
- Pub/Sub, Streams (lightweight messaging)
- Leaderboards (sorted sets)

---

## 13. Queues & Background Jobs

Move slow or unreliable work out of the request cycle: emails, image processing, reports, webhooks.

```js
// BullMQ (Redis-backed)
const { Queue, Worker } = require('bullmq');
const connection = { host: '127.0.0.1', port: 6379 };

const emailQueue = new Queue('email', { connection });

// Producer (in API)
await emailQueue.add('welcome', { userId: 42 }, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 2000 },
  removeOnComplete: 1000,
  jobId: `welcome-42`,           // dedupe / idempotency
});

// Consumer (separate process)
new Worker('email', async (job) => {
  await sendWelcomeEmail(job.data.userId);
}, { connection, concurrency: 10 });
```

**Design rules**

- Jobs must be **idempotent**; delivery is typically **at-least-once**.
- Use retries with backoff and a **dead-letter queue** for poison messages.
- Keep payloads small (IDs, not entire objects).
- Monitor queue depth and job age.

**Outbox pattern** ⚫: write the event to an `outbox` table in the *same DB transaction* as the business change; a relay publishes it to the queue. This prevents "DB committed but message lost" inconsistencies.

---

## 14. Rate Limiting & Resilience Patterns

### 14.1 Rate-limiting algorithms 🔴

| Algorithm | Behaviour |
|---|---|
| Fixed window | Simple; allows bursts at window edges |
| Sliding window | Smoother; more memory/compute |
| Token bucket | Allows controlled bursts; refill at constant rate |
| Leaky bucket | Smooths output rate |

**Distributed fixed-window limiter with Redis**

```js
async function rateLimit(key, limit, windowSec) {
  const k = `rl:${key}:${Math.floor(Date.now() / 1000 / windowSec)}`;
  const count = await redis.incr(k);
  if (count === 1) await redis.expire(k, windowSec);
  return { allowed: count <= limit, remaining: Math.max(0, limit - count) };
}

app.use(async (req, res, next) => {
  const { allowed, remaining } = await rateLimit(req.ip, 100, 60);
  res.set('X-RateLimit-Remaining', String(remaining));
  if (!allowed) return res.status(429).set('Retry-After', '60').json({ error: 'Too many requests' });
  next();
});
```

(For production use `rate-limiter-flexible` or `express-rate-limit` with a Redis store.)

### 14.2 Circuit breaker

```js
class CircuitBreaker {
  constructor(fn, { failureThreshold = 5, resetMs = 10_000 } = {}) {
    Object.assign(this, { fn, failureThreshold, resetMs, failures: 0, state: 'CLOSED', openedAt: 0 });
  }
  async call(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.openedAt < this.resetMs) throw new Error('Circuit open');
      this.state = 'HALF_OPEN';
    }
    try {
      const r = await this.fn(...args);
      this.failures = 0; this.state = 'CLOSED';
      return r;
    } catch (e) {
      if (++this.failures >= this.failureThreshold || this.state === 'HALF_OPEN') {
        this.state = 'OPEN'; this.openedAt = Date.now();
      }
      throw e;
    }
  }
}
```

(Library: `opossum`.)

### 14.3 Related patterns

- **Timeouts** on every outbound call (default is often "forever").
- **Bulkheads**: separate pools/queues so one slow dependency doesn't starve everything.
- **Graceful degradation**: serve cached/partial data when a dependency fails.
- **Load shedding**: reject early with 503 when overloaded (`toobusy-js`, event-loop-delay threshold).

---

## 15. Real-Time: WebSockets

### 15.1 Raw `ws` 🟡

```js
const { WebSocketServer } = require('ws');
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  ws.isAlive = true;
  ws.on('pong', () => (ws.isAlive = true));
  ws.on('message', (data) => {
    for (const client of wss.clients) {
      if (client !== ws && client.readyState === ws.OPEN) client.send(data.toString());
    }
  });
});

// Heartbeat: detect dead connections
setInterval(() => {
  for (const ws of wss.clients) {
    if (!ws.isAlive) { ws.terminate(); continue; }
    ws.isAlive = false;
    ws.ping();
  }
}, 30_000);
```

### 15.2 Scaling WebSockets 🔴

- Connections are **stateful** and pinned to one server. Messages to users on another instance need a **pub/sub backplane** (Redis Pub/Sub, Socket.IO Redis adapter, NATS).
- Use sticky sessions only if the library needs them (Socket.IO long-polling fallback).
- Authenticate during the handshake (token in query/header/first message), not after.
- Handle **backpressure**: check `ws.bufferedAmount` before sending to slow clients.
- Rate-limit messages per connection.

### 15.3 Alternatives

| Need | Choice |
|---|---|
| Server → client only | **Server-Sent Events (SSE)**: simpler, auto-reconnect, works over plain HTTP |
| Two-way, low latency | WebSockets |
| Fallbacks, rooms, acks | Socket.IO |

---

## 16. Architecture & Design Patterns

### 16.1 Layered (a.k.a. clean-ish) architecture 🔴

```
HTTP layer      routes / controllers   → parse, validate, call service, shape response
Service layer   business logic         → framework-agnostic, unit-testable
Repository      data access            → SQL/ORM details hidden here
Infrastructure  db, cache, queue, mail → adapters behind interfaces
```

```js
// repository
class UserRepo {
  constructor(db) { this.db = db; }
  findByEmail(email) { return this.db.query('SELECT * FROM users WHERE email=$1', [email]); }
}

// service (depends on abstractions, not on Express)
class AuthService {
  constructor({ users, hasher, tokens }) { Object.assign(this, { users, hasher, tokens }); }
  async login(email, password) {
    const user = await this.users.findByEmail(email);
    if (!user || !(await this.hasher.verify(user.hash, password))) throw new AuthError();
    return this.tokens.issue(user);
  }
}

// composition root (wire dependencies in ONE place)
const authService = new AuthService({ users: new UserRepo(pool), hasher: argon2, tokens });
```

**Dependency injection** gives you easy mocking in tests. Frameworks: NestJS, `awilix`, `tsyringe`, or manual wiring (often enough).

### 16.2 Patterns worth knowing

| Pattern | Use |
|---|---|
| Repository | Isolate persistence |
| Factory / Builder | Complex object creation |
| Strategy | Swappable algorithms (payment gateways) |
| Observer (EventEmitter) | Decoupled notifications |
| Middleware / Chain of responsibility | Express pipeline |
| Adapter | Wrap third-party APIs |
| CQRS | Separate read and write models |
| Event sourcing | Store events as source of truth |
| Saga | Distributed transactions with compensating actions |

### 16.3 Twelve-Factor essentials

Config in env · stateless processes · logs to stdout · disposable processes (fast start, graceful stop) · dev/prod parity · backing services as attached resources.

### 16.4 Monolith first

Start with a **modular monolith** (clear module boundaries, one deployable). Split into services only when team size, scaling needs or deployment independence justify the operational cost.

---

## 17. Advanced Testing

### 17.1 Test pyramid

```
        /\      E2E (few, slow)             Playwright, Cypress
       /  \     Integration (some)          Supertest + real DB (Testcontainers)
      /____\    Unit (many, fast)           Jest / Vitest / node:test
```

### 17.2 Integration tests with a real database 🔴

```js
const { PostgreSqlContainer } = require('@testcontainers/postgresql');

let container, pool;
beforeAll(async () => {
  container = await new PostgreSqlContainer().start();
  pool = new Pool({ connectionString: container.getConnectionUri() });
  await runMigrations(pool);
}, 60_000);
afterAll(async () => { await pool.end(); await container.stop(); });

beforeEach(async () => { await pool.query('TRUNCATE users RESTART IDENTITY CASCADE'); });

test('POST /users creates a user', async () => {
  const res = await request(makeApp({ pool })).post('/users').send({ email: 'a@b.com', password: 'x'.repeat(12) });
  expect(res.status).toBe(201);
  const { rows } = await pool.query('SELECT * FROM users');
  expect(rows).toHaveLength(1);
});
```

### 17.3 Mocking and time 🟡

```js
jest.useFakeTimers();
const p = retryOperation();
await jest.advanceTimersByTimeAsync(5000);
await p;

// Spy on a method
const spy = jest.spyOn(emailClient, 'send').mockResolvedValue({ ok: true });
await service.register(dto);
expect(spy).toHaveBeenCalledWith(expect.objectContaining({ to: dto.email }));
```

### 17.4 Node's built-in test runner

```js
// test/math.test.js
const test = require('node:test');
const assert = require('node:assert/strict');

test('adds', () => assert.equal(1 + 2, 3));
test('async', async () => assert.rejects(fail(), /boom/));
```

```bash
node --test            # run
node --test --watch
node --test --experimental-test-coverage
```

### 17.5 Quality practices

- Test **behaviour**, not implementation details.
- Make `app` creation a function (`makeApp(deps)`) so tests inject fakes.
- **Contract tests** (Pact) for service-to-service APIs.
- **Property-based testing** (`fast-check`) for tricky logic.
- Run tests in CI with coverage thresholds; use **mutation testing** (Stryker) to check test quality.

---

## 18. TypeScript with Node

```bash
npm i -D typescript tsx @types/node
npx tsc --init
```

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

```ts
// Typed Express handler
import type { Request, Response } from 'express';

interface CreateUserBody { email: string; password: string }

export async function createUser(
  req: Request<{}, {}, CreateUserBody>,
  res: Response,
) { /* ... */ }

// Runtime validation AND static types from ONE source
import { z } from 'zod';
const User = z.object({ id: z.string().uuid(), email: z.string().email() });
type User = z.infer<typeof User>;

// Discriminated unions for results
type Result<T> = { ok: true; value: T } | { ok: false; error: Error };
```

**Dev workflow:** `tsx watch src/server.ts` · **Build:** `tsc` · **Run:** `node dist/server.js`.
(Recent Node versions can also strip types natively for simple TS files.)

---

## 19. ESM, CJS Interop & Module Internals

### 19.1 What `require` does 🔴

1. **Resolve** the path (core module? relative? `node_modules` lookup walking up directories?)
2. **Load** the file
3. **Wrap** it in a function: `(function (exports, require, module, __filename, __dirname) { ... })`
4. **Execute**, then **cache** `module.exports` in `require.cache`

### 19.2 Circular dependencies

```js
// a.js
exports.loaded = false;
const b = require('./b');
exports.loaded = true;

// b.js
const a = require('./a');          // gets PARTIAL exports of a: { loaded: false }
console.log('a.loaded =', a.loaded);
```

Fix by restructuring (extract shared code), lazy `require` inside functions, or dependency injection.

### 19.3 `exports` vs `module.exports`

```js
exports = { a: 1 };            // BROKEN: only reassigns the local variable
module.exports = { a: 1 };     // works
exports.a = 1;                 // works (mutates the shared object)
```

### 19.4 ESM differences

| | CJS | ESM |
|---|---|---|
| Binding | Copy of values | **Live bindings** |
| `__dirname` | Available | Use `import.meta.dirname` (Node 20.11+) or `fileURLToPath(import.meta.url)` |
| Top-level `await` | No | Yes |
| Loading | Sync | Async, static analysis |
| `require` of ESM | Historically not allowed | Supported in recent Node versions for sync ESM graphs |

```js
// Dynamic import works in both systems
const { default: chalk } = await import('chalk');

// package.json "exports" for dual packages
{
  "exports": {
    ".": { "import": "./dist/index.mjs", "require": "./dist/index.cjs" }
  }
}
```

---

## 20. Docker, CI/CD & Production Deployment

### 20.1 Production Dockerfile (multi-stage) 🟡

```dockerfile
# ---- build ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

# ---- runtime ----
FROM node:22-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD node -e "fetch('http://localhost:3000/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]
```

**Docker tips:** `.dockerignore` (node_modules, .git, .env) · run as non-root · pin base image versions · use `dumb-init`/`tini` or `--init` so signals are forwarded correctly · copy `package*.json` first to leverage layer cache.

### 20.2 Health checks and graceful shutdown 🔴

```js
let shuttingDown = false;

app.get('/healthz', (req, res) => res.send('ok'));                    // liveness
app.get('/readyz', async (req, res) => {                              // readiness
  if (shuttingDown) return res.sendStatus(503);
  try { await pool.query('SELECT 1'); res.send('ready'); }
  catch { res.sendStatus(503); }
});

const server = app.listen(PORT);

async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info({ signal }, 'shutting down');
  setTimeout(() => process.exit(1), 25_000).unref();      // hard deadline

  server.close(async () => {                               // stop accepting, finish in-flight
    await Promise.allSettled([pool.end(), redis.quit(), queue.close()]);
    process.exit(0);
  });
  server.closeIdleConnections?.();
}
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

### 20.3 CI pipeline outline (GitHub Actions)

```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: pass }
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready --health-interval 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage
      - run: npm audit --omit=dev --audit-level=high
```

### 20.4 Deployment strategies

| Strategy | Idea |
|---|---|
| Rolling | Replace instances gradually |
| Blue/green | Two environments, switch traffic |
| Canary | Send small % of traffic to the new version |
| Feature flags | Decouple deploy from release |

### 20.5 Behind a reverse proxy

Run Nginx/Envoy/ALB in front for TLS termination, compression, static files, buffering, and load balancing. Set `app.set('trust proxy', ...)` correctly and tune `server.keepAliveTimeout` to be **greater** than the load balancer's idle timeout to avoid sporadic 502s.

---

## 21. Observability: Logs, Metrics, Tracing

### 21.1 Structured logging 🔴

```js
const pino = require('pino');
const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization', 'password', '*.token'],
});

app.use(require('pino-http')({ logger, genReqId: () => crypto.randomUUID() }));
req.log.info({ userId, orderId }, 'order placed');
```

Rules: JSON logs to **stdout** · include a correlation/request ID · never log secrets or PII · use levels consistently.

### 21.2 Metrics (RED / USE)

- **RED** for services: **R**ate, **E**rrors, **D**uration.
- **USE** for resources: **U**tilization, **S**aturation, **E**rrors.

```js
const client = require('prom-client');
client.collectDefaultMetrics();                          // event loop lag, heap, GC…

const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on('finish', () => end({ method: req.method, route: req.route?.path ?? 'unknown', status: res.statusCode }));
  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

Avoid **high-cardinality labels** (user IDs, raw URLs).

### 21.3 Distributed tracing

Use **OpenTelemetry** (`@opentelemetry/sdk-node`, auto-instrumentations) to trace a request across services. Propagate `traceparent` headers; export to Jaeger, Tempo, Datadog, etc.

### 21.4 Alerting

Alert on **symptoms** (error rate, latency SLO burn) more than causes (CPU). Define **SLIs/SLOs** (e.g. 99.9% of requests < 500 ms).

---

## 22. Microservices & Messaging

### 22.1 Communication styles ⚫

| Style | Examples | Trade-off |
|---|---|---|
| Synchronous | REST, gRPC | Simple, but temporal coupling and cascading failures |
| Asynchronous | Kafka, RabbitMQ, SQS, NATS | Decoupled and resilient, but eventual consistency and harder debugging |

### 22.2 Delivery semantics

- **At-most-once**: may lose messages.
- **At-least-once**: may duplicate, so consumers must be **idempotent**.
- **Exactly-once**: really "at-least-once + idempotent processing" (or transactional features in specific systems).

### 22.3 Saga (choreography example)

```
OrderService   → emits OrderCreated
PaymentService ← consumes; charges → emits PaymentSucceeded | PaymentFailed
InventoryService ← on PaymentSucceeded reserves stock → emits StockReserved | StockFailed
OrderService   ← on any failure runs COMPENSATION (cancel order, refund)
```

### 22.4 API gateway and BFF

Gateway handles auth, rate limiting, routing, aggregation. A **Backend-for-Frontend** tailors responses per client type (web, mobile).

### 22.5 gRPC basics

Protocol Buffers contracts, HTTP/2, streaming, strong typing. Use `@grpc/grpc-js` and `@grpc/proto-loader`.

### 22.6 Distributed-systems realities

- Networks fail; every call needs **timeout + retry + idempotency**.
- **CAP**: during a partition choose consistency or availability.
- Clocks differ between machines; don't rely on wall-clock ordering.
- Version your APIs and events; make schema changes backward compatible.

---

## 23. Hard Interview Questions

**Internals**

1. Walk through exactly what happens, phase by phase, when a request arrives at an Express server, including libuv's role.
2. Why does `setImmediate` always run before `setTimeout(0)` inside an I/O callback, but not at top level?
3. Which operations use the libuv thread pool, and which don't? What happens with 5 concurrent `pbkdf2` calls and the default pool size?
4. How does Node achieve concurrency without threads for network I/O?
5. What are `process.nextTick` starvation and its fix?
6. Explain the module wrapper function and what `require.cache` does.
7. How are ES module live bindings different from CJS value copies?

**Performance and scale**

8. Your API's p99 latency spikes but CPU averages 30%. How do you investigate? (event-loop delay, GC pauses, thread-pool saturation, downstream slowness, connection pool waits)
9. How would you handle CPU-intensive tasks without freezing the server?
10. Compare cluster, worker threads, and horizontal scaling with containers.
11. What is backpressure? Show how a missing `drain` handler leads to OOM.
12. How do you find a memory leak in production?
13. How would you design to handle 100k concurrent WebSocket connections?

**Design and reliability**

14. Design a rate limiter for a distributed API. Compare algorithms and failure modes (Redis down: fail open or closed?).
15. How do you guarantee an email is sent exactly once after an order is saved? (outbox, idempotency, at-least-once)
16. Explain cache stampede and three ways to prevent it.
17. How do you implement zero-downtime deployments and DB migrations?
18. How would you implement graceful shutdown in Kubernetes? (readiness flips, `preStop`, SIGTERM, drain, deadline)
19. Optimistic vs pessimistic locking: when to use which?
20. How do you make a retry safe for a payment endpoint?

**Security**

21. How can a JWT-based system revoke access? What are the trade-offs?
22. Explain prototype pollution with an exploit and defence.
23. How would you prevent SSRF in a webhook-sending feature?
24. What is ReDoS and how do you mitigate it in Node specifically?

**Output prediction snippet**

```js
console.log('A');
setTimeout(() => console.log('B'), 0);
Promise.resolve().then(() => console.log('C'));
process.nextTick(() => console.log('D'));
(async () => { console.log('E'); await null; console.log('F'); })();
console.log('G');
```

<details><summary>Answer</summary>

`A E G D C F B`. Sync code first (A, E, G; `await` pauses the async function after E). Then `nextTick` (D), then microtasks in queue order (C, then F), then the timer (B).

</details>

---

## 24. Practice Problems

### 🟡 Moderate

1. **Concurrency limiter:** implement `pLimit(n)` returning a function that wraps async tasks so at most `n` run at once. Add tests proving the limit holds.
2. **Retry middleware:** build an HTTP client wrapper with timeout, retries with jittered backoff, and `Retry-After` header support.
3. **JWT auth service:** register/login/refresh/logout with refresh-token rotation and hashed refresh tokens in the DB.
4. **LRU cache:** implement with a `Map` (O(1) get/set/evict), TTL support, and a hit/miss stats method.
5. **CSV importer:** stream a 2 GB CSV into a database in batches of 1,000 rows using `pipeline`, with constant memory.

### 🔴 Difficult

6. **Worker pool:** build a generic pool of N worker threads with a task queue, timeouts, crash recovery and graceful close. Benchmark against single-threaded execution.
7. **Rate limiter:** implement sliding-window and token-bucket limiters backed by Redis using Lua scripts for atomicity. Load test them with `autocannon`.
8. **Idempotent payments API:** `POST /payments` with an `Idempotency-Key` header. Concurrent duplicate requests must produce one charge and identical responses.
9. **Job queue from scratch:** persistent queue in PostgreSQL using `SELECT ... FOR UPDATE SKIP LOCKED`, with retries, visibility timeout, and dead-letter table.
10. **Memory-leak lab:** write an app with 3 deliberate leaks (global cache, listener leak, timer leak). Find and fix each using heap snapshots; document evidence.
11. **Graceful shutdown harness:** prove with a test that in-flight requests complete and new connections are refused after SIGTERM.

### ⚫ Expert

12. **Mini Express:** implement routing with params, middleware chain, error middleware, and body parsing on top of `http` only. Then add async error propagation.
13. **Event-loop profiler:** build a tool that reports event-loop delay percentiles and attributes slow ticks to code using `async_hooks`/`perf_hooks` and sampled stack traces.
14. **Distributed lock with fencing tokens** using Redis and show (with a failing test) why naive locks break under process pauses.
15. **Outbox + relay:** implement the transactional outbox with a relay that publishes to RabbitMQ/Kafka and a consumer deduplicating by message ID.
16. **Real-time collaborative counter/editor** with WebSockets, horizontal scaling through Redis pub/sub, and conflict resolution (e.g., CRDT counter or OT-lite).

---

## 25. Capstone Projects

Pick one; aim for production-quality practices (tests, CI, Docker, observability, docs).

| Project | Key skills exercised |
|---|---|
| **URL shortener at scale** | Base62 IDs, Redis caching, rate limiting, analytics via queue, read/write split |
| **Real-time chat platform** | WebSockets, Redis backplane, presence, message history pagination, auth |
| **E-commerce API** | Transactions, inventory locking, idempotent checkout, outbox, payments webhooks |
| **Job/notification service** | BullMQ or custom queue, retries, DLQ, scheduling, admin dashboard |
| **File processing pipeline** | Streaming uploads to S3, worker threads for image/video work, progress via SSE |
| **API gateway** | Routing, JWT validation, rate limiting, circuit breaker, request tracing |

**Grading rubric (suggested)**

| Area | Weight |
|---|---|
| Correctness & tests | 25% |
| Architecture & code quality | 20% |
| Performance (benchmarks, profiling evidence) | 15% |
| Security | 15% |
| Reliability (shutdown, retries, health checks) | 15% |
| Documentation & DX (README, diagrams, scripts) | 10% |

---

## 26. Resources

- Node.js docs: https://nodejs.org/docs · Guides: https://nodejs.org/en/learn
- Event loop explained (official): "The Node.js Event Loop, Timers, and process.nextTick()" in the Node docs
- Node Best Practices: https://github.com/goldbergyoni/nodebestpractices
- Node Security Best Practices: https://nodejs.org/en/learn/getting-started/security-best-practices
- OWASP Cheat Sheet Series: https://cheatsheetseries.owasp.org
- Designing Data-Intensive Applications (Martin Kleppmann): the standard book for the databases, caching, messaging and distributed-systems topics above
- Clinic.js: https://clinicjs.org · OpenTelemetry JS: https://opentelemetry.io/docs/languages/js/
- 12-Factor App: https://12factor.net

---

### Suggested study plan (8 weeks)

| Week | Focus |
|---|---|
| 1 | Event loop, async patterns, error handling |
| 2 | Streams, backpressure, worker threads, cluster |
| 3 | Auth, security hardening, validation |
| 4 | Databases: pooling, transactions, indexing, migrations |
| 5 | Redis caching, queues, rate limiting, resilience |
| 6 | Architecture, testing (integration/contract), TypeScript |
| 7 | Docker, CI/CD, observability, graceful shutdown |
| 8 | Capstone project + mock interviews |

## License

MIT
