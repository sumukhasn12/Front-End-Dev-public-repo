# Express.js + MongoDB — Step-by-Step Student Guide

Build a complete, secure **REST API** (users, authentication, tasks) using **Node.js, Express, MongoDB and Mongoose**. Every step has: *what you are doing*, *the code*, and *how to check that it works*.

**Who this is for:** students who know basic JavaScript and have used Node.js a little. No MongoDB or Express experience needed.

**What you will build:** a **Task Manager API**

| Feature | Details |
|---|---|
| Register / Login | Passwords hashed with bcrypt, JWT tokens |
| Protected routes | Only logged-in users can reach their tasks |
| Task CRUD | Create, read, update, delete tasks |
| Filtering, search, sorting, pagination | `GET /api/tasks?status=todo&search=exam&sort=-dueDate&page=2` |
| Statistics | Aggregation pipeline: tasks per status |
| Clean error handling | One consistent JSON error format |
| Admin-only route | Role-based authorization |

---

## Table of Contents

- [Part 0: Concepts](#part-0-concepts)
- [Part 1: Set Up MongoDB](#part-1-set-up-mongodb)
- [Part 2: Learn MongoDB Commands (mongosh)](#part-2-learn-mongodb-commands-mongosh)
- [Part 3: Create the Express Project](#part-3-create-the-express-project)
- [Part 4: Connect Express to MongoDB](#part-4-connect-express-to-mongodb)
- [Part 5: Models (Mongoose Schemas)](#part-5-models-mongoose-schemas)
- [Part 6: Error Handling Foundation](#part-6-error-handling-foundation)
- [Part 7: Authentication (Register and Login)](#part-7-authentication-register-and-login)
- [Part 8: Task CRUD](#part-8-task-crud)
- [Part 9: Filtering, Search, Sorting, Pagination](#part-9-filtering-search-sorting-pagination)
- [Part 10: Validation with Zod](#part-10-validation-with-zod)
- [Part 11: Relationships and populate](#part-11-relationships-and-populate)
- [Part 12: Aggregation (Stats)](#part-12-aggregation-stats)
- [Part 13: Role-Based Access (Admin)](#part-13-role-based-access-admin)
- [Part 14: Security Middleware](#part-14-security-middleware)
- [Part 15: Test the API (curl / Postman)](#part-15-test-the-api-curl--postman)
- [Part 16: Automated Tests](#part-16-automated-tests)
- [Part 17: Indexes and Performance](#part-17-indexes-and-performance)
- [Part 18: Deploy](#part-18-deploy)
- [Troubleshooting](#troubleshooting)
- [Cheat Sheets](#cheat-sheets)
- [Assignments](#assignments)
- [Interview Questions](#interview-questions)

---

## Part 0: Concepts

### 0.1 What is Express?

A minimal web framework on top of Node's `http` module. It gives you **routing**, **middleware**, and helpers for requests/responses.

### 0.2 What is MongoDB?

A **NoSQL document database**. Data is stored as **BSON** (binary JSON) documents.

| SQL term | MongoDB term |
|---|---|
| Database | Database |
| Table | **Collection** |
| Row | **Document** |
| Column | **Field** |
| Primary key | `_id` (auto-generated **ObjectId**) |
| JOIN | `$lookup` / `populate()` / embedding |

A document:

```json
{
  "_id": "665f1c2e9b1d8a3f4c2a1b00",
  "title": "Finish assignment",
  "status": "todo",
  "tags": ["college", "urgent"],
  "dueDate": "2026-12-01T00:00:00Z"
}
```

### 0.3 What is Mongoose?

An **ODM** (Object Data Modeling) library. It adds on top of the MongoDB driver:

- **Schemas**: structure, types, defaults
- **Validation**: required, min/max, enum, custom
- **Middleware (hooks)**: e.g. hash password before save
- **Convenient queries**: `Task.find().sort().limit()`

### 0.4 Request lifecycle in our API

```
Client (Postman / browser / React app)
   │  HTTP request
   ▼
Express app
   ├─ global middleware: helmet → cors → json parser → logger → rate limiter
   ├─ router:  /api/tasks
   │     ├─ protect middleware  (verify JWT, load user)
   │     ├─ validate middleware (check body with Zod)
   │     └─ controller          (business logic)
   │              └─ Mongoose model ──► MongoDB
   ├─ 404 handler
   └─ error handler  (one place to format all errors)
   ▼
JSON response
```

### 0.5 REST conventions we follow

| Action | Method | URL | Success code |
|---|---|---|---|
| Register | POST | `/api/auth/register` | 201 |
| Login | POST | `/api/auth/login` | 200 |
| Current user | GET | `/api/auth/me` | 200 |
| List tasks | GET | `/api/tasks` | 200 |
| Task stats | GET | `/api/tasks/stats` | 200 |
| Get one | GET | `/api/tasks/:id` | 200 |
| Create | POST | `/api/tasks` | 201 |
| Update | PATCH | `/api/tasks/:id` | 200 |
| Delete | DELETE | `/api/tasks/:id` | 204 |
| List users (admin) | GET | `/api/users` | 200 |

---

## Part 1: Set Up MongoDB

Pick **Option A (Atlas, cloud)** if you want zero installation, or **Option B (local)**.

### Option A: MongoDB Atlas (free cloud database)

1. Go to https://www.mongodb.com/atlas and create a free account.
2. Click **Create** a deployment → choose the **free (M0)** tier → pick a region close to you → create.
3. **Database Access** → *Add New Database User* → choose username + password (save them!) → give *Read and write to any database*.
4. **Network Access** → *Add IP Address* → for learning, choose **Allow access from anywhere** (`0.0.0.0/0`). In production, allow only your server's IP.
5. **Database** → click **Connect** on your cluster → **Drivers** → copy the connection string:

```
mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/taskmanager?retryWrites=true&w=majority
```

6. Replace `<username>` and `<password>`. If the password has special characters (`@ : / ?`), **URL-encode** them (`@` → `%40`). Add the database name (`taskmanager`) after the `/`.

### Option B: Install MongoDB locally

- **Windows / macOS / Linux:** download *MongoDB Community Server* from https://www.mongodb.com/try/download/community and follow the installer.
- **macOS (Homebrew):**

```bash
brew tap mongodb/brew
brew install mongodb-community
brew services start mongodb-community
```

- **Ubuntu:** follow the official docs for your version, then `sudo systemctl start mongod`.
- **Docker (works everywhere):**

```bash
docker run -d --name mongo -p 27017:27017 -v mongo_data:/data/db mongo:7
```

Local connection string:

```
mongodb://127.0.0.1:27017/taskmanager
```

### Tools

| Tool | Purpose |
|---|---|
| **MongoDB Compass** | GUI to browse/query data (https://www.mongodb.com/products/tools/compass) |
| **mongosh** | Command-line shell |
| **Postman / Thunder Client** | Test API requests |

### Check it works

```bash
mongosh "mongodb://127.0.0.1:27017"
> db.runCommand({ ping: 1 })
{ ok: 1 }
```

For Atlas: `mongosh "your-atlas-connection-string"`.

---

## Part 2: Learn MongoDB Commands (mongosh)

Practise before using Mongoose. Open `mongosh`:

```js
use school            // switches to (and lazily creates) database "school"
```

### 2.1 Create (insert)

```js
db.students.insertOne({ name: "Asha", age: 20, course: "CSE", skills: ["js", "node"] })

db.students.insertMany([
  { name: "Ravi",  age: 22, course: "ECE", skills: ["c"] },
  { name: "Meena", age: 21, course: "CSE", skills: ["python", "sql"] },
  { name: "Kiran", age: 19, course: "IT",  skills: [] }
])
```

### 2.2 Read (find)

```js
db.students.find()                                   // all
db.students.find({ course: "CSE" })                  // filter
db.students.findOne({ name: "Asha" })                // first match
db.students.find({}, { name: 1, age: 1, _id: 0 })    // projection (choose fields)
db.students.find().sort({ age: -1 })                 // sort desc (1 = asc)
db.students.find().skip(1).limit(2)                  // pagination
db.students.countDocuments({ course: "CSE" })
```

### 2.3 Query operators

| Operator | Meaning | Example |
|---|---|---|
| `$eq`, `$ne` | equal / not equal | `{ course: { $ne: "IT" } }` |
| `$gt`, `$gte`, `$lt`, `$lte` | comparison | `{ age: { $gte: 20 } }` |
| `$in`, `$nin` | in / not in list | `{ course: { $in: ["CSE", "IT"] } }` |
| `$and`, `$or`, `$nor`, `$not` | logic | `{ $or: [{ age: { $lt: 20 } }, { course: "ECE" }] }` |
| `$exists` | field present | `{ phone: { $exists: false } }` |
| `$regex` | pattern | `{ name: { $regex: "^A", $options: "i" } }` |
| `$all`, `$size`, `$elemMatch` | arrays | `{ skills: { $all: ["js", "node"] } }` |

### 2.4 Update

```js
db.students.updateOne({ name: "Asha" }, { $set: { age: 21 } })
db.students.updateMany({ course: "CSE" }, { $inc: { age: 1 } })
db.students.updateOne({ name: "Asha" }, { $push: { skills: "mongodb" } })
db.students.updateOne({ name: "Asha" }, { $addToSet: { skills: "js" } })  // no duplicates
db.students.updateOne({ name: "Asha" }, { $pull: { skills: "node" } })
db.students.updateOne({ name: "Zed" }, { $set: { age: 18 } }, { upsert: true })  // insert if missing
```

| Update operator | Purpose |
|---|---|
| `$set` / `$unset` | set / remove a field |
| `$inc` | add to a number |
| `$push` / `$addToSet` / `$pull` | array changes |
| `$rename` | rename a field |

> Forgetting `$set` replaces nothing in `updateOne` (it errors), but `replaceOne` replaces the *whole* document. Be careful which you use.

### 2.5 Delete

```js
db.students.deleteOne({ name: "Kiran" })
db.students.deleteMany({ age: { $lt: 18 } })
db.students.drop()                // delete the whole collection
```

### 2.6 Aggregation (preview)

```js
db.students.aggregate([
  { $match: { age: { $gte: 20 } } },
  { $group: { _id: "$course", total: { $sum: 1 }, avgAge: { $avg: "$age" } } },
  { $sort: { total: -1 } }
])
```

### 2.7 Indexes (preview)

```js
db.students.createIndex({ name: 1 })
db.students.createIndex({ email: 1 }, { unique: true })
db.students.getIndexes()
db.students.find({ name: "Asha" }).explain("executionStats")   // look for IXSCAN, not COLLSCAN
```

### Exercise 2

Using a `library` database, insert 6 books (`title, author, year, price, tags[]`). Then: find books after 2015; find books priced between 200 and 500; add a tag to one book; increase all prices by 10%; delete books older than 1990; group books by author with count and average price.

---

## Part 3: Create the Express Project

### Step 3.1 Check Node

```bash
node -v        # 18+ (20 or 22 LTS recommended)
npm -v
```

### Step 3.2 Initialize

```bash
mkdir express-mongo-api
cd express-mongo-api
npm init -y
git init
```

### Step 3.3 Install packages

```bash
npm install express mongoose dotenv bcryptjs jsonwebtoken zod cors helmet morgan express-rate-limit
npm install --save-dev nodemon
```

| Package | Why |
|---|---|
| `express` | Web framework |
| `mongoose` | MongoDB ODM |
| `dotenv` | Load `.env` into `process.env` |
| `bcryptjs` | Hash passwords (pure JS, no build tools needed) |
| `jsonwebtoken` | Create/verify JWTs |
| `zod` | Request validation |
| `cors` | Allow browser apps from other origins |
| `helmet` | Secure HTTP headers |
| `morgan` | Request logging |
| `express-rate-limit` | Slow down brute-force attacks |
| `nodemon` (dev) | Auto-restart on file changes |

### Step 3.4 Edit `package.json` scripts

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "test": "jest --runInBand"
  }
}
```

### Step 3.5 Folder structure

```bash
mkdir -p src/{config,models,controllers,routes,middleware,utils,validators}
touch src/app.js src/server.js
touch .env .env.example .gitignore
```

```
express-mongo-api/
├── src/
│   ├── config/
│   │   └── db.js
│   ├── models/
│   │   ├── User.js
│   │   └── Task.js
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── task.controller.js
│   │   └── user.controller.js
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── task.routes.js
│   │   └── user.routes.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── error.js
│   │   └── validate.js
│   ├── validators/
│   │   └── schemas.js
│   ├── utils/
│   │   ├── AppError.js
│   │   └── asyncHandler.js
│   ├── app.js          ← builds the Express app (no listen)
│   └── server.js       ← connects DB and starts listening
├── .env
├── .env.example
├── .gitignore
└── package.json
```

> **Why `app.js` and `server.js` are separate:** tests can import `app` without opening a port or connecting to the real DB.

### Step 3.6 `.gitignore`

```
node_modules/
.env
*.log
coverage/
```

### Step 3.7 `.env` (never commit) and `.env.example` (commit this)

```ini
NODE_ENV=development
PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/taskmanager
JWT_SECRET=replace-with-a-long-random-string-at-least-32-chars
JWT_EXPIRES_IN=7d
```

Generate a strong secret:

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

`.env.example` has the same keys with fake values (so teammates know what to set).

### Step 3.8 First server: `src/app.js`

```js
const express = require('express');

const app = express();

app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

module.exports = app;
```

### Step 3.9 `src/server.js`

```js
require('dotenv').config();
const app = require('./app');

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

### ✅ Check

```bash
npm run dev
# in another terminal:
curl http://localhost:5000/health
# {"status":"ok","uptime":3.12}
```

---

## Part 4: Connect Express to MongoDB

### Step 4.1 `src/config/db.js`

```js
const mongoose = require('mongoose');

async function connectDB(uri) {
  if (!uri) throw new Error('MONGO_URI is not defined in .env');

  mongoose.connection.on('connected', () => console.log('MongoDB connected'));
  mongoose.connection.on('error', (err) => console.error('MongoDB error:', err.message));
  mongoose.connection.on('disconnected', () => console.warn('MongoDB disconnected'));

  await mongoose.connect(uri, {
    serverSelectionTimeoutMS: 5000,   // fail fast if DB unreachable
  });
}

module.exports = connectDB;
```

### Step 4.2 Update `src/server.js`

Start listening **only after** the DB is connected, and shut down cleanly.

```js
require('dotenv').config();
const mongoose = require('mongoose');
const app = require('./app');
const connectDB = require('./config/db');

const PORT = process.env.PORT || 5000;

async function start() {
  try {
    await connectDB(process.env.MONGO_URI);
    const server = app.listen(PORT, () =>
      console.log(`Server running on http://localhost:${PORT}`)
    );

    // Graceful shutdown (Ctrl+C or platform stop signal)
    const shutdown = async (signal) => {
      console.log(`\n${signal} received. Closing...`);
      server.close(async () => {
        await mongoose.connection.close();
        process.exit(0);
      });
    };
    process.on('SIGINT', () => shutdown('SIGINT'));
    process.on('SIGTERM', () => shutdown('SIGTERM'));
  } catch (err) {
    console.error('Failed to start:', err.message);
    process.exit(1);
  }
}

start();

process.on('unhandledRejection', (err) => {
  console.error('Unhandled rejection:', err);
  process.exit(1);
});
```

### ✅ Check

`npm run dev` should print `MongoDB connected` and `Server running...`. If it fails, see [Troubleshooting](#troubleshooting).

---

## Part 5: Models (Mongoose Schemas)

### 5.1 Schema basics

```js
const schema = new mongoose.Schema({
  title: { type: String, required: [true, 'Title is required'], trim: true, maxlength: 100 },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
}, { timestamps: true });     // adds createdAt, updatedAt
```

**Common schema options**

| Option | Purpose |
|---|---|
| `required` | Must exist |
| `unique` | Creates a unique **index** (not a validator) |
| `default` | Default value |
| `enum` | Allowed values |
| `min` / `max` / `minlength` / `maxlength` | Limits |
| `trim`, `lowercase`, `uppercase` | Auto-clean strings |
| `select: false` | Hide by default in queries |
| `match` | Regex validation |
| `validate` | Custom validator function |

### 5.2 User model: `src/models/User.js`

```js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'Name is required'],
      trim: true,
      minlength: [2, 'Name must be at least 2 characters'],
      maxlength: 50,
    },
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
      trim: true,
      match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email'],
    },
    password: {
      type: String,
      required: [true, 'Password is required'],
      minlength: [8, 'Password must be at least 8 characters'],
      select: false,               // not returned unless .select('+password')
    },
    role: {
      type: String,
      enum: ['user', 'admin'],
      default: 'user',
    },
  },
  {
    timestamps: true,
    toJSON: {
      transform(doc, ret) {
        delete ret.password;
        delete ret.__v;
        return ret;
      },
    },
  }
);

// Hash password before saving (only when it changed)
userSchema.pre('save', async function () {
  if (!this.isModified('password')) return;
  this.password = await bcrypt.hash(this.password, 12);
});

// Instance method: compare a plain password with the stored hash
userSchema.methods.comparePassword = function (candidate) {
  return bcrypt.compare(candidate, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

**Why `select: false`?** So a forgotten `.find()` never leaks password hashes. In login we ask for it explicitly with `.select('+password')`.

### 5.3 Task model: `src/models/Task.js`

```js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: [true, 'Title is required'],
      trim: true,
      maxlength: [100, 'Title cannot exceed 100 characters'],
    },
    description: { type: String, trim: true, maxlength: 1000, default: '' },
    status: {
      type: String,
      enum: { values: ['todo', 'in-progress', 'done'], message: '{VALUE} is not a valid status' },
      default: 'todo',
    },
    priority: {
      type: String,
      enum: ['low', 'medium', 'high'],
      default: 'medium',
    },
    dueDate: { type: Date },
    tags: { type: [String], default: [] },
    user: {
      type: mongoose.Schema.Types.ObjectId,   // reference to a User document
      ref: 'User',
      required: true,
      index: true,
    },
  },
  { timestamps: true }
);

// Compound index: speeds up "my tasks filtered by status, newest first"
taskSchema.index({ user: 1, status: 1, createdAt: -1 });

module.exports = mongoose.model('Task', taskSchema);
```

### ✅ Check (optional quick script)

Create `scratch.js` in the project root:

```js
require('dotenv').config();
const mongoose = require('mongoose');
const User = require('./src/models/User');

(async () => {
  await mongoose.connect(process.env.MONGO_URI);
  const u = await User.create({ name: 'Test', email: 'T@Example.com', password: 'password123' });
  console.log(u.toJSON());            // email lowercased, no password in output
  await User.deleteMany({});
  await mongoose.disconnect();
})();
```

Run `node scratch.js`, then delete the file.

---

## Part 6: Error Handling Foundation

Build this **before** controllers so every controller stays clean.

### Step 6.1 `src/utils/AppError.js`

```js
class AppError extends Error {
  constructor(message, statusCode = 500, details) {
    super(message);
    this.statusCode = statusCode;
    this.details = details;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}
module.exports = AppError;
```

### Step 6.2 `src/utils/asyncHandler.js`

Wraps async controllers so rejected promises reach the error handler (works in Express 4 and 5).

```js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

module.exports = asyncHandler;
```

### Step 6.3 `src/middleware/error.js`

```js
const AppError = require('../utils/AppError');

function notFound(req, res, next) {
  next(new AppError(`Route not found: ${req.method} ${req.originalUrl}`, 404));
}

// eslint-disable-next-line no-unused-vars
function errorHandler(err, req, res, next) {
  let status = err.statusCode || 500;
  let message = err.message || 'Internal Server Error';
  let details = err.details;

  // Mongoose validation error (schema rules failed)
  if (err.name === 'ValidationError' && err.errors) {
    status = 400;
    message = 'Validation failed';
    details = Object.values(err.errors).map((e) => ({ field: e.path, message: e.message }));
  }

  // Mongoose CastError (e.g. invalid ObjectId in URL)
  if (err.name === 'CastError') {
    status = 400;
    message = `Invalid value for ${err.path}`;
  }

  // MongoDB duplicate key (unique index violated)
  if (err.code === 11000) {
    status = 409;
    const field = Object.keys(err.keyValue || {})[0] || 'field';
    message = `${field} already exists`;
  }

  // JWT errors
  if (err.name === 'JsonWebTokenError') { status = 401; message = 'Invalid token'; }
  if (err.name === 'TokenExpiredError') { status = 401; message = 'Token expired, please log in again'; }

  // Malformed JSON body
  if (err.type === 'entity.parse.failed') { status = 400; message = 'Malformed JSON body'; }

  if (status >= 500) {
    console.error(err);
    if (process.env.NODE_ENV === 'production') message = 'Internal Server Error';
  }

  res.status(status).json({
    success: false,
    message,
    ...(details && { details }),
    ...(process.env.NODE_ENV === 'development' && status >= 500 && { stack: err.stack }),
  });
}

module.exports = { notFound, errorHandler };
```

---

## Part 7: Authentication (Register and Login)

### 7.1 How it works

```
REGISTER: client sends name/email/password
          → server hashes password (bcrypt) → saves user → returns JWT

LOGIN:    client sends email/password
          → server finds user, compares hash → returns JWT

LATER:    client sends header  Authorization: Bearer <token>
          → protect middleware verifies token → loads user → req.user
```

A **JWT** has 3 parts (`header.payload.signature`). The payload is *readable by anyone* (base64), but can't be modified without the secret. **Never put passwords or sensitive data in it.**

### Step 7.1 `src/controllers/auth.controller.js`

```js
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const AppError = require('../utils/AppError');
const asyncHandler = require('../utils/asyncHandler');

function signToken(userId) {
  return jwt.sign({ sub: userId }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  });
}

exports.register = asyncHandler(async (req, res) => {
  const { name, email, password } = req.body;

  // NOTE: we pick fields explicitly, so users cannot self-assign role: 'admin'
  const user = await User.create({ name, email, password });

  res.status(201).json({
    success: true,
    token: signToken(user._id),
    user,                              // toJSON strips the password
  });
});

exports.login = asyncHandler(async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ email }).select('+password');

  // Same message for "no user" and "wrong password": don't reveal which emails exist
  if (!user || !(await user.comparePassword(password))) {
    throw new AppError('Invalid email or password', 401);
  }

  res.json({ success: true, token: signToken(user._id), user });
});

exports.me = asyncHandler(async (req, res) => {
  res.json({ success: true, user: req.user });
});
```

### Step 7.2 `src/middleware/auth.js`

```js
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const AppError = require('../utils/AppError');
const asyncHandler = require('../utils/asyncHandler');

// Must be logged in
exports.protect = asyncHandler(async (req, res, next) => {
  const header = req.headers.authorization;
  if (!header || !header.startsWith('Bearer ')) {
    throw new AppError('Authentication required', 401);
  }
  const token = header.split(' ')[1];

  const decoded = jwt.verify(token, process.env.JWT_SECRET);   // throws on invalid/expired

  const user = await User.findById(decoded.sub);
  if (!user) throw new AppError('User no longer exists', 401);

  req.user = user;
  next();
});

// Must have one of these roles (use AFTER protect)
exports.restrictTo = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user.role)) {
    return next(new AppError('You do not have permission to do this', 403));
  }
  next();
};
```

### Step 7.3 `src/routes/auth.routes.js`

```js
const router = require('express').Router();
const { register, login, me } = require('../controllers/auth.controller');
const { protect } = require('../middleware/auth');

router.post('/register', register);
router.post('/login', login);
router.get('/me', protect, me);

module.exports = router;
```

### Step 7.4 Wire it into `src/app.js`

Replace `src/app.js` with:

```js
const express = require('express');
const authRoutes = require('./routes/auth.routes');
const { notFound, errorHandler } = require('./middleware/error');

const app = express();

app.use(express.json({ limit: '10kb' }));

app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.use('/api/auth', authRoutes);

app.use(notFound);        // after all routes
app.use(errorHandler);    // very last

module.exports = app;
```

### ✅ Check

```bash
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Asha","email":"asha@example.com","password":"password123"}'
```

Expected: `201` with a `token` and the `user` (no password). Register the same email again → `409 email already exists`. Then log in and call `/api/auth/me` with the token:

```bash
curl http://localhost:5000/api/auth/me -H "Authorization: Bearer YOUR_TOKEN"
```

Open **Compass** and check the `users` collection: the stored `password` should look like `$2a$12$...` (a hash).

---

## Part 8: Task CRUD

### Step 8.1 `src/controllers/task.controller.js` (basic version)

```js
const Task = require('../models/Task');
const AppError = require('../utils/AppError');
const asyncHandler = require('../utils/asyncHandler');

// Only these fields may be set from the request body (prevents mass assignment)
const ALLOWED = ['title', 'description', 'status', 'priority', 'dueDate', 'tags'];
const pick = (obj, keys) =>
  Object.fromEntries(keys.filter((k) => obj[k] !== undefined).map((k) => [k, obj[k]]));

// CREATE
exports.createTask = asyncHandler(async (req, res) => {
  const task = await Task.create({ ...pick(req.body, ALLOWED), user: req.user._id });
  res.status(201).json({ success: true, task });
});

// READ ALL (we'll upgrade this in Part 9)
exports.getTasks = asyncHandler(async (req, res) => {
  const tasks = await Task.find({ user: req.user._id }).sort('-createdAt');
  res.json({ success: true, count: tasks.length, tasks });
});

// READ ONE: note the { _id, user } filter: you can only read YOUR tasks
exports.getTask = asyncHandler(async (req, res) => {
  const task = await Task.findOne({ _id: req.params.id, user: req.user._id });
  if (!task) throw new AppError('Task not found', 404);
  res.json({ success: true, task });
});

// UPDATE (partial)
exports.updateTask = asyncHandler(async (req, res) => {
  const task = await Task.findOneAndUpdate(
    { _id: req.params.id, user: req.user._id },
    { $set: pick(req.body, ALLOWED) },
    { new: true, runValidators: true }          // return updated doc + apply schema rules
  );
  if (!task) throw new AppError('Task not found', 404);
  res.json({ success: true, task });
});

// DELETE
exports.deleteTask = asyncHandler(async (req, res) => {
  const task = await Task.findOneAndDelete({ _id: req.params.id, user: req.user._id });
  if (!task) throw new AppError('Task not found', 404);
  res.status(204).send();
});
```

**Key lessons**

- Always include `user: req.user._id` in the filter. Without it, user A could read/edit user B's task by guessing an ID (**IDOR**).
- Return **404** (not 403) for other people's tasks, so IDs don't leak.
- `runValidators: true` is required for updates; otherwise Mongoose skips schema validation on `findOneAndUpdate`.
- `{ new: true }` returns the document *after* the update.

### Step 8.2 `src/routes/task.routes.js`

```js
const router = require('express').Router();
const {
  createTask, getTasks, getTask, updateTask, deleteTask,
} = require('../controllers/task.controller');
const { protect } = require('../middleware/auth');

router.use(protect);                // every task route requires login

router.route('/')
  .get(getTasks)
  .post(createTask);

router.route('/:id')
  .get(getTask)
  .patch(updateTask)
  .delete(deleteTask);

module.exports = router;
```

### Step 8.3 Register the routes in `src/app.js`

```js
const taskRoutes = require('./routes/task.routes');
// ...
app.use('/api/auth', authRoutes);
app.use('/api/tasks', taskRoutes);
```

### ✅ Check

```bash
TOKEN="paste-your-token"

curl -X POST http://localhost:5000/api/tasks \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"title":"Learn Express","priority":"high","tags":["study"]}'

curl http://localhost:5000/api/tasks -H "Authorization: Bearer $TOKEN"
```

Try: no token → `401`; invalid id like `/api/tasks/abc` → `400 Invalid value for _id`; a valid-looking but unknown id → `404`.

---

## Part 9: Filtering, Search, Sorting, Pagination

Replace `getTasks` in `task.controller.js`:

```js
// Escape user text before putting it in a regex (prevents ReDoS / regex injection)
const escapeRegex = (s) => s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');

const SORTABLE = ['createdAt', 'dueDate', 'priority', 'title', 'status'];

exports.getTasks = asyncHandler(async (req, res) => {
  const { status, priority, search, tag, sort = '-createdAt' } = req.query;

  // 1. FILTER: always scoped to the logged-in user
  const filter = { user: req.user._id };

  // typeof checks block NoSQL injection like ?status[$ne]=x
  if (typeof status === 'string') filter.status = status;
  if (typeof priority === 'string') filter.priority = priority;
  if (typeof tag === 'string') filter.tags = tag;
  if (typeof search === 'string' && search.trim()) {
    const rx = new RegExp(escapeRegex(search.trim()), 'i');
    filter.$or = [{ title: rx }, { description: rx }];
  }

  // 2. SORT: whitelist fields ("-dueDate" = descending)
  const sortField = typeof sort === 'string' ? sort.replace(/^-/, '') : '';
  const sortBy = SORTABLE.includes(sortField) ? sort : '-createdAt';

  // 3. PAGINATION
  const page = Math.max(parseInt(req.query.page, 10) || 1, 1);
  const limit = Math.min(Math.max(parseInt(req.query.limit, 10) || 10, 1), 100);
  const skip = (page - 1) * limit;

  // Run both queries at the same time
  const [tasks, total] = await Promise.all([
    Task.find(filter).sort(sortBy).skip(skip).limit(limit),
    Task.countDocuments(filter),
  ]);

  res.json({
    success: true,
    count: tasks.length,
    pagination: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
      hasNext: page * limit < total,
      hasPrev: page > 1,
    },
    tasks,
  });
});
```

### Example requests

```
GET /api/tasks?status=todo
GET /api/tasks?priority=high&sort=-dueDate
GET /api/tasks?search=exam&page=2&limit=5
GET /api/tasks?tag=college
```

### Understanding `skip`/`limit`

Page 3, limit 10 → `skip = 20`, returns documents 21–30. For *very* large collections `skip` becomes slow (the DB still walks the skipped docs). The alternative, **cursor (keyset) pagination**:

```js
// Next page: items older than the last one seen
Task.find({ user, createdAt: { $lt: lastCreatedAt } }).sort('-createdAt').limit(10);
```

---

## Part 10: Validation with Zod

Mongoose validates at the database layer. **Zod** validates early at the API edge, giving clear errors and rejecting unknown fields before touching the DB.

### Step 10.1 `src/validators/schemas.js`

```js
const { z } = require('zod');

const status = z.enum(['todo', 'in-progress', 'done']);
const priority = z.enum(['low', 'medium', 'high']);

exports.registerSchema = z.object({
  name: z.string().trim().min(2).max(50),
  email: z.string().trim().toLowerCase().email(),
  password: z.string().min(8).max(128),
}).strict();                                   // reject unknown keys (e.g. "role")

exports.loginSchema = z.object({
  email: z.string().trim().toLowerCase().email(),
  password: z.string().min(1),
}).strict();

const taskFields = {
  title: z.string().trim().min(1).max(100),
  description: z.string().trim().max(1000),
  status,
  priority,
  dueDate: z.coerce.date(),                    // "2026-12-01" → Date
  tags: z.array(z.string().trim().min(1).max(30)).max(10),
};

exports.createTaskSchema = z.object({
  title: taskFields.title,
  description: taskFields.description.optional(),
  status: status.optional(),
  priority: priority.optional(),
  dueDate: taskFields.dueDate.optional(),
  tags: taskFields.tags.optional(),
}).strict();

exports.updateTaskSchema = z.object(taskFields).partial().strict()
  .refine((o) => Object.keys(o).length > 0, { message: 'Provide at least one field to update' });
```

### Step 10.2 `src/middleware/validate.js`

```js
const AppError = require('../utils/AppError');

// validate(schema) → checks req.body, replaces it with the cleaned data
module.exports = (schema) => (req, res, next) => {
  const result = schema.safeParse(req.body);
  if (!result.success) {
    const details = result.error.issues.map((i) => ({
      field: i.path.join('.'),
      message: i.message,
    }));
    return next(new AppError('Validation failed', 400, details));
  }
  req.body = result.data;
  next();
};
```

### Step 10.3 Use it in the routes

`auth.routes.js`:

```js
const validate = require('../middleware/validate');
const { registerSchema, loginSchema } = require('../validators/schemas');

router.post('/register', validate(registerSchema), register);
router.post('/login', validate(loginSchema), login);
```

`task.routes.js`:

```js
const validate = require('../middleware/validate');
const { createTaskSchema, updateTaskSchema } = require('../validators/schemas');

router.route('/')
  .get(getTasks)
  .post(validate(createTaskSchema), createTask);

router.route('/:id')
  .get(getTask)
  .patch(validate(updateTaskSchema), updateTask)
  .delete(deleteTask);
```

### ✅ Check

- `POST /api/auth/register` with `{"name":"A","email":"bad","password":"1"}` → `400` with a `details` array.
- Register with `"role":"admin"` in the body → `400` (unknown key). Good: no privilege escalation.
- `PATCH /api/tasks/:id` with `{}` → `400 Provide at least one field`.

---

## Part 11: Relationships and populate

### 11.1 Embedding vs referencing

| | **Embed** (nested in the same doc) | **Reference** (store `_id`) |
|---|---|---|
| Example | `address` inside `user` | `task.user` → user |
| Read speed | One read | Extra query / `$lookup` |
| Use when | Data is read together, small, owned by parent | Data is shared, large, or grows unbounded |
| Limit | Doc ≤ **16 MB** | None |
| Duplicated data | Possible | No |

Rule of thumb: **model for how your app reads data.**

### 11.2 `populate()`: replace an ObjectId with the real document

```js
const task = await Task.findOne({ _id: req.params.id, user: req.user._id })
  .populate('user', 'name email');      // second arg = fields to include
```

Result:

```json
{ "title": "Learn Express", "user": { "_id": "...", "name": "Asha", "email": "asha@example.com" } }
```

`populate` runs a **second query** behind the scenes. For many docs use it deliberately and select only needed fields.

### 11.3 Virtuals and cascading delete

When a user is deleted, remove their tasks. Add to `User.js` (before `module.exports`):

```js
userSchema.pre('deleteOne', { document: true, query: false }, async function () {
  await mongoose.model('Task').deleteMany({ user: this._id });
});
```

Use `await user.deleteOne()` on a document for this hook to fire.

---

## Part 12: Aggregation (Stats)

The **aggregation pipeline** passes documents through stages: `$match` → `$group` → `$sort` → `$project`…

### Step 12.1 Add to `task.controller.js`

```js
exports.getStats = asyncHandler(async (req, res) => {
  // NOTE: aggregate() does NOT auto-cast strings to ObjectId, so use req.user._id (already ObjectId)
  const [byStatus, byPriority, overdue] = await Promise.all([
    Task.aggregate([
      { $match: { user: req.user._id } },
      { $group: { _id: '$status', count: { $sum: 1 } } },
      { $sort: { _id: 1 } },
    ]),
    Task.aggregate([
      { $match: { user: req.user._id } },
      { $group: { _id: '$priority', count: { $sum: 1 } } },
    ]),
    Task.countDocuments({
      user: req.user._id,
      status: { $ne: 'done' },
      dueDate: { $lt: new Date() },
    }),
  ]);

  res.json({ success: true, stats: { byStatus, byPriority, overdue } });
});
```

### Step 12.2 Route (place **before** `/:id`!)

```js
router.get('/stats', getStats);          // must be above router.route('/:id')
router.route('/').get(getTasks).post(validate(createTaskSchema), createTask);
```

> **Why order matters:** Express matches routes top-to-bottom. If `/:id` comes first, the word `stats` is treated as an id and causes a `CastError`.

### Useful stages

| Stage | Purpose |
|---|---|
| `$match` | Filter (put early to use indexes) |
| `$group` | Group and compute (`$sum`, `$avg`, `$min`, `$max`, `$push`) |
| `$project` | Choose/compute fields |
| `$sort`, `$limit`, `$skip` | Order and page |
| `$lookup` | Left outer join with another collection |
| `$unwind` | Split array into one doc per element |
| `$facet` | Multiple pipelines in one query |
| `$count` | Count docs at that stage |

Example: tasks per tag:

```js
Task.aggregate([
  { $match: { user: req.user._id } },
  { $unwind: '$tags' },
  { $group: { _id: '$tags', total: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 5 },
]);
```

---

## Part 13: Role-Based Access (Admin)

### Step 13.1 `src/controllers/user.controller.js`

```js
const User = require('../models/User');
const Task = require('../models/Task');
const AppError = require('../utils/AppError');
const asyncHandler = require('../utils/asyncHandler');

exports.getUsers = asyncHandler(async (req, res) => {
  const users = await User.find().sort('-createdAt');
  res.json({ success: true, count: users.length, users });
});

exports.deleteUser = asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new AppError('User not found', 404);
  if (user._id.equals(req.user._id)) throw new AppError('You cannot delete yourself', 400);

  await Task.deleteMany({ user: user._id });
  await user.deleteOne();
  res.status(204).send();
});
```

### Step 13.2 `src/routes/user.routes.js`

```js
const router = require('express').Router();
const { getUsers, deleteUser } = require('../controllers/user.controller');
const { protect, restrictTo } = require('../middleware/auth');

router.use(protect, restrictTo('admin'));

router.get('/', getUsers);
router.delete('/:id', deleteUser);

module.exports = router;
```

Register: `app.use('/api/users', require('./routes/user.routes'));`

### Step 13.3 Create the first admin

Users cannot register as admin (good!). Promote one manually in `mongosh`:

```js
use taskmanager
db.users.updateOne({ email: "asha@example.com" }, { $set: { role: "admin" } })
```

Log in again (or just use the same token; the role is read from the DB on each request) and call `GET /api/users`. A normal user gets `403`.

---

## Part 14: Security Middleware

Update `src/app.js` to its final form:

```js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const rateLimit = require('express-rate-limit');

const authRoutes = require('./routes/auth.routes');
const taskRoutes = require('./routes/task.routes');
const userRoutes = require('./routes/user.routes');
const { notFound, errorHandler } = require('./middleware/error');

const app = express();

// If behind a proxy/load balancer (Render, Heroku, Nginx), so req.ip is correct
app.set('trust proxy', 1);

app.use(helmet());                                              // secure headers
app.use(cors({ origin: process.env.CORS_ORIGIN?.split(',') || '*' }));
app.use(express.json({ limit: '10kb' }));                       // cap body size
if (process.env.NODE_ENV !== 'test') app.use(morgan('dev'));    // request logs

// Strict limiter for login/register (brute-force protection)
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: process.env.NODE_ENV === 'test' ? 1000 : 20,
  standardHeaders: true,
  legacyHeaders: false,
  message: { success: false, message: 'Too many attempts, try again later' },
});

// Gentler limiter for everything else under /api
const apiLimiter = rateLimit({ windowMs: 60 * 1000, limit: 100, standardHeaders: true, legacyHeaders: false });

app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.use('/api', apiLimiter);
app.use('/api/auth', authLimiter, authRoutes);
app.use('/api/tasks', taskRoutes);
app.use('/api/users', userRoutes);

app.use(notFound);
app.use(errorHandler);

module.exports = app;
```

Add to `.env`: `CORS_ORIGIN=http://localhost:3000` (comma-separate multiple origins).

### Security checklist

- [x] Passwords hashed (bcrypt, cost 12)
- [x] JWT secret in env, long and random
- [x] Body size limited, unknown fields rejected (Zod `.strict()`)
- [x] Mass assignment prevented (`pick` and explicit fields)
- [x] NoSQL injection guarded (`typeof` checks, schema validation)
- [x] Ownership enforced in every task query (no IDOR)
- [x] Rate limiting, Helmet, CORS
- [x] Errors never leak stack traces in production
- [ ] HTTPS (handled by your hosting platform / reverse proxy)
- [ ] Refresh tokens / logout everywhere (advanced exercise)

---

## Part 15: Test the API (curl / Postman)

### 15.1 Postman setup

1. Create a **Collection** "Task Manager".
2. Create an **Environment** with `baseUrl = http://localhost:5000` and `token` (empty).
3. In **Login** request → *Tests* tab:

```js
pm.environment.set("token", pm.response.json().token);
```

4. In the collection → *Authorization* → Type **Bearer Token** → value `{{token}}`.

### 15.2 Full test script (curl)

```bash
BASE=http://localhost:5000/api

# 1. Register
curl -s -X POST $BASE/auth/register -H "Content-Type: application/json" \
  -d '{"name":"Asha","email":"asha@example.com","password":"password123"}'

# 2. Login → copy token
curl -s -X POST $BASE/auth/login -H "Content-Type: application/json" \
  -d '{"email":"asha@example.com","password":"password123"}'

TOKEN="paste-token-here"
AUTH="Authorization: Bearer $TOKEN"

# 3. Create tasks
curl -s -X POST $BASE/tasks -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"title":"Prepare for exam","priority":"high","dueDate":"2026-12-10","tags":["college"]}'

curl -s -X POST $BASE/tasks -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"title":"Buy groceries","tags":["home"]}'

# 4. List / filter / search / paginate
curl -s "$BASE/tasks" -H "$AUTH"
curl -s "$BASE/tasks?priority=high&sort=-dueDate" -H "$AUTH"
curl -s "$BASE/tasks?search=exam" -H "$AUTH"
curl -s "$BASE/tasks?page=1&limit=1" -H "$AUTH"

# 5. Update / delete (replace ID)
curl -s -X PATCH $BASE/tasks/TASK_ID -H "$AUTH" -H "Content-Type: application/json" -d '{"status":"done"}'
curl -s -X DELETE $BASE/tasks/TASK_ID -H "$AUTH" -i

# 6. Stats
curl -s $BASE/tasks/stats -H "$AUTH"
```

### 15.3 Test matrix (do all of these)

| # | Request | Expected |
|---|---|---|
| 1 | Register valid | 201 + token |
| 2 | Register same email | 409 |
| 3 | Register short password | 400 + details |
| 4 | Register with `role: admin` | 400 |
| 5 | Login wrong password | 401 |
| 6 | `GET /tasks` without token | 401 |
| 7 | `GET /tasks` with garbage token | 401 Invalid token |
| 8 | Create task without title | 400 |
| 9 | `GET /tasks/abc` | 400 invalid id |
| 10 | `GET /tasks/<valid unknown id>` | 404 |
| 11 | User B reads User A's task id | 404 |
| 12 | `PATCH` with invalid status | 400 |
| 13 | `GET /users` as normal user | 403 |
| 14 | `GET /tasks?status[$ne]=done` | Ignored (still scoped, no injection) |
| 15 | Send invalid JSON | 400 Malformed JSON |

---

## Part 16: Automated Tests

### Step 16.1 Install

```bash
npm install --save-dev jest supertest mongodb-memory-server
```

`mongodb-memory-server` starts a temporary in-memory MongoDB (downloads a MongoDB binary the first time, so it needs internet once).

### Step 16.2 `package.json`

```json
"jest": { "testEnvironment": "node" }
```

### Step 16.3 `tests/api.test.js`

```js
process.env.NODE_ENV = 'test';
process.env.JWT_SECRET = 'test-secret-test-secret-test-secret';
process.env.JWT_EXPIRES_IN = '1h';

const request = require('supertest');
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');
const app = require('../src/app');

let mongod;

beforeAll(async () => {
  mongod = await MongoMemoryServer.create();
  await mongoose.connect(mongod.getUri());
}, 120000);

afterAll(async () => {
  await mongoose.disconnect();
  await mongod.stop();
});

afterEach(async () => {
  for (const key in mongoose.connection.collections) {
    await mongoose.connection.collections[key].deleteMany({});
  }
});

const user = { name: 'Asha', email: 'asha@example.com', password: 'password123' };

async function registerAndGetToken(u = user) {
  const res = await request(app).post('/api/auth/register').send(u);
  return res.body.token;
}

describe('Auth', () => {
  test('registers a user and hides password', async () => {
    const res = await request(app).post('/api/auth/register').send(user);
    expect(res.status).toBe(201);
    expect(res.body.token).toBeDefined();
    expect(res.body.user.password).toBeUndefined();
  });

  test('rejects duplicate email', async () => {
    await request(app).post('/api/auth/register').send(user);
    const res = await request(app).post('/api/auth/register').send(user);
    expect(res.status).toBe(409);
  });

  test('rejects wrong password', async () => {
    await request(app).post('/api/auth/register').send(user);
    const res = await request(app).post('/api/auth/login').send({ email: user.email, password: 'wrongpass' });
    expect(res.status).toBe(401);
  });
});

describe('Tasks', () => {
  test('requires authentication', async () => {
    const res = await request(app).get('/api/tasks');
    expect(res.status).toBe(401);
  });

  test('creates and lists tasks', async () => {
    const token = await registerAndGetToken();
    const create = await request(app)
      .post('/api/tasks')
      .set('Authorization', `Bearer ${token}`)
      .send({ title: 'Write tests' });
    expect(create.status).toBe(201);
    expect(create.body.task.status).toBe('todo');

    const list = await request(app).get('/api/tasks').set('Authorization', `Bearer ${token}`);
    expect(list.body.count).toBe(1);
  });

  test("user cannot access another user's task", async () => {
    const tokenA = await registerAndGetToken();
    const tokenB = await registerAndGetToken({ name: 'Ravi', email: 'ravi@example.com', password: 'password123' });

    const created = await request(app)
      .post('/api/tasks').set('Authorization', `Bearer ${tokenA}`).send({ title: 'Private' });

    const res = await request(app)
      .get(`/api/tasks/${created.body.task._id}`).set('Authorization', `Bearer ${tokenB}`);
    expect(res.status).toBe(404);
  });

  test('validates input', async () => {
    const token = await registerAndGetToken();
    const res = await request(app).post('/api/tasks').set('Authorization', `Bearer ${token}`).send({});
    expect(res.status).toBe(400);
  });
});
```

Run: `npm test`.

---

## Part 17: Indexes and Performance

### 17.1 What is an index?

A sorted data structure (B-tree) that lets MongoDB find documents without scanning the whole collection.

- Without index: **COLLSCAN** (reads every doc) → O(n)
- With index: **IXSCAN** → O(log n)

Trade-off: indexes speed reads but cost disk/RAM and slow writes slightly. Index what you query on.

### 17.2 See it in mongosh

```js
db.tasks.find({ user: ObjectId("..."), status: "todo" }).sort({ createdAt: -1 }).explain("executionStats")
```

Check `winningPlan.stage` (want `IXSCAN`) and `totalDocsExamined` ≈ `nReturned`.

### 17.3 Index types

| Type | Example | Use |
|---|---|---|
| Single field | `{ email: 1 }` | Lookups by one field |
| Compound | `{ user: 1, status: 1, createdAt: -1 }` | Filter + sort (order matters!) |
| Unique | `{ email: 1 }, { unique: true }` | No duplicates |
| Text | `{ title: "text", description: "text" }` | Full-text search |
| TTL | `{ expiresAt: 1 }, { expireAfterSeconds: 0 }` | Auto-delete (sessions, OTPs) |
| Partial | `{ ... }, { partialFilterExpression: {...} }` | Index a subset |

**Compound index rule (ESR):** **E**quality fields first, then **S**ort fields, then **R**ange fields.

### 17.4 Other performance habits

- Use `.lean()` for read-only queries (returns plain objects, much faster): `Task.find(f).lean()`.
- Use `.select('title status')` to fetch only needed fields.
- Always paginate; never return unbounded lists.
- Avoid unbounded arrays inside documents.
- In production set `mongoose.set('autoIndex', false)` and create indexes via migrations (building indexes on startup can be heavy on big collections).
- Reuse the single Mongoose connection (never connect per request).

### 17.5 Transactions (multi-document atomicity)

Needs a replica set (Atlas has one by default).

```js
const session = await mongoose.startSession();
try {
  await session.withTransaction(async () => {
    await Account.updateOne({ _id: from }, { $inc: { balance: -amt } }, { session });
    await Account.updateOne({ _id: to },   { $inc: { balance:  amt } }, { session });
  });
} finally {
  session.endSession();
}
```

Single-document operations are already atomic, so good schema design often avoids needing transactions.

---

## Part 18: Deploy

### 18.1 Prepare

1. Use **Atlas** for the database. In Network Access allow your host's IP (or `0.0.0.0/0` for simple hosting, with a strong DB password).
2. Make sure `package.json` has `"start": "node src/server.js"`.
3. Never commit `.env`.
4. Add `"engines": { "node": ">=20" }` to `package.json`.

### 18.2 Deploy on Render (example)

1. Push code to GitHub.
2. Render → **New Web Service** → connect your repo.
3. Build command: `npm install` · Start command: `npm start`.
4. **Environment** → add `NODE_ENV=production`, `MONGO_URI`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `CORS_ORIGIN`.
5. Deploy → test `https://your-app.onrender.com/health`.

(Railway, Fly.io, Heroku-style platforms, and a VPS with PM2 + Nginx work similarly.)

### 18.3 Pre-launch checklist

- [ ] `NODE_ENV=production`
- [ ] Strong, unique `JWT_SECRET`
- [ ] CORS limited to your frontend's origin
- [ ] Atlas backups enabled; DB user has least privilege
- [ ] Logs visible (stdout), health endpoint checked
- [ ] `npm audit` clean of high issues

---

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `MongooseServerSelectionError: connect ECONNREFUSED 127.0.0.1:27017` | Local MongoDB isn't running. Start `mongod` / `brew services start mongodb-community` / `docker start mongo` |
| `MongooseServerSelectionError` with Atlas | IP not allowed (Network Access), wrong password, or no internet/DNS block |
| `bad auth : Authentication failed` | Wrong DB username/password; URL-encode special characters in the password |
| `querySrv ENOTFOUND` | DNS issue with `mongodb+srv://`; try another network or the non-SRV connection string from Atlas |
| `MONGO_URI is not defined` | `.env` missing/misnamed, or `dotenv.config()` not called first in `server.js` |
| `Cannot POST /api/...` | Wrong URL/method or router not mounted in `app.js` |
| `req.body` is `undefined` | Missing `app.use(express.json())` or missing `Content-Type: application/json` header |
| `E11000 duplicate key` | Unique field repeated. Handled as 409 by our error handler |
| `CastError: Cast to ObjectId failed` | Bad id format in URL, or `/stats` placed after `/:id` |
| Password not hashed / login always fails | You used `findByIdAndUpdate` to set password (it skips `save` hooks). Use `doc.password = ...; await doc.save()` |
| Update ignores validation | Missing `runValidators: true` |
| `jwt malformed` | Token missing, truncated, or `Bearer ` prefix not stripped |
| Changes not applying | Not using `nodemon`; restart server |
| `EADDRINUSE :::5000` | Port in use. Change `PORT` or stop the other process (`lsof -i :5000`) |
| Mongoose "buffering timed out" | Queries ran before/without a successful connection; connect first, then start the server |

---

## Cheat Sheets

### Mongoose queries

```js
Model.create(doc)
Model.insertMany([docs])
Model.find(filter, projection)
Model.findOne(filter)
Model.findById(id)
Model.countDocuments(filter)
Model.exists(filter)
Model.findOneAndUpdate(filter, update, { new: true, runValidators: true })
Model.updateMany(filter, update)
Model.findOneAndDelete(filter)
Model.deleteMany(filter)
Model.aggregate([...])

// chainable modifiers
.select('a b -c').sort('-createdAt').skip(n).limit(n).populate('ref').lean()
```

### Express essentials

```js
app.use(mw)                          // global middleware
app.use('/path', router)             // mount router
router.route('/').get(h).post(h)     // chain methods
req.params.id   req.query.x   req.body   req.headers   req.user (ours)
res.status(201).json({...})   res.sendStatus(204)   res.redirect(url)
next()            // continue to next middleware
next(err)         // jump to the error handler
```

### HTTP status codes used

`200` OK · `201` Created · `204` No Content · `400` Bad Request · `401` Unauthenticated · `403` Forbidden · `404` Not Found · `409` Conflict · `429` Too Many Requests · `500` Server Error

### Git commit plan (one commit per Part)

```
feat: project setup and health route
feat: mongodb connection
feat: user and task models
feat: error handling
feat: auth register/login/me
feat: task CRUD
feat: filtering, sorting, pagination
feat: zod validation
feat: stats aggregation
feat: admin routes
chore: security middleware
test: api tests
```

---

## Assignments

### Level 1: Guided

1. Add a `completedAt` field that is automatically set when a task's status becomes `done` (and cleared if it moves back).
2. Add `GET /api/tasks/overdue` returning tasks past `dueDate` that aren't done.
3. Add `PATCH /api/auth/password` (change password; require the current password).
4. Add a `DELETE /api/tasks` that deletes all *completed* tasks of the logged-in user and returns how many were deleted.

### Level 2: Independent

5. **Projects:** add a `Project` model (`name`, `owner`). Tasks can belong to a project (`project: ObjectId`). Add CRUD for projects and filter tasks by project. Deleting a project must handle its tasks.
6. **Comments:** embed or reference comments on tasks (`text`, `author`, `createdAt`). Decide embed vs reference and justify.
7. **Soft delete:** add `deletedAt`; hide soft-deleted tasks from normal queries, add restore endpoint (hint: a Mongoose `pre(/^find/)` hook).
8. **Text search:** replace the regex search with a MongoDB **text index** and sort results by relevance (`$meta: "textScore"`).
9. **Task stats by month** using `$group` with `$month`/`$year` on `createdAt`.

### Level 3: Challenge

10. **Refresh tokens:** short-lived access token (15 min) + httpOnly refresh cookie, stored hashed, rotated on use, revoked on logout.
11. **Shared tasks:** let a task have `collaborators: [userId]` with `viewer`/`editor` permissions. Update every query and test the permission matrix.
12. **Email verification and password reset** with expiring, hashed one-time tokens (TTL index to auto-clean).
13. **File uploads:** attach files to tasks using `multer` (+ S3/Cloudinary), validating type and size.
14. **Swagger/OpenAPI docs** for every endpoint using `swagger-ui-express`.
15. **Rate-limit store in Redis** and caching of `GET /tasks/stats` with invalidation on writes.

### Mini-project ideas (reuse the same patterns)

Blog platform · Expense tracker · Library management · Student attendance system · E-commerce backend (products, cart, orders) · Event booking system

---

## Interview Questions

**Express**

1. What is middleware? In what order does Express run it? What does `next()` / `next(err)` do?
2. Difference between `app.use()` and `app.get()`?
3. How does Express identify an error-handling middleware?
4. How do `req.params`, `req.query` and `req.body` differ?
5. Why must `/stats` be defined before `/:id`?
6. How do you structure a large Express app?

**MongoDB**

7. SQL vs NoSQL: when would you choose MongoDB?
8. What is `_id`? What is an ObjectId made of? (timestamp + random + counter)
9. Embedding vs referencing: trade-offs?
10. What is an index? What is a compound index and why does field order matter?
11. What does `explain()` show? What's COLLSCAN vs IXSCAN?
12. What is the aggregation framework? Write a pipeline that gets the top 3 categories by total sales.
13. What are replica sets and sharding? (availability/redundancy vs horizontal scaling)
14. Are MongoDB operations atomic? (single-document yes; multi-document needs transactions)
15. `skip/limit` vs cursor pagination?

**Mongoose**

16. What is a schema vs a model? What are `pre`/`post` hooks?
17. Why do `findOneAndUpdate` calls need `runValidators: true`?
18. What does `populate()` do and what's its cost?
19. What does `lean()` do?
20. `unique: true`: validator or index?

**Security and design**

21. How do you store passwords? Why bcrypt and not SHA-256?
22. How can NoSQL injection happen (`{"email": {"$ne": null}}`), and how do you prevent it?
23. What is IDOR/BOLA and how did we prevent it in this project?
24. What's mass assignment? Show a vulnerable example.
25. Where should JWTs be stored on the client, and what are the trade-offs (localStorage vs httpOnly cookie)?

---

## Final Project Structure (for reference)

```
express-mongo-api/
├── src/
│   ├── config/db.js
│   ├── controllers/{auth,task,user}.controller.js
│   ├── middleware/{auth,error,validate}.js
│   ├── models/{User,Task}.js
│   ├── routes/{auth,task,user}.routes.js
│   ├── utils/{AppError,asyncHandler}.js
│   ├── validators/schemas.js
│   ├── app.js
│   └── server.js
├── tests/api.test.js
├── .env            (not committed)
├── .env.example
├── .gitignore
└── package.json
```

## Useful Links

- Express: https://expressjs.com
- Mongoose: https://mongoosejs.com/docs
- MongoDB Manual: https://www.mongodb.com/docs/manual
- MongoDB University (free courses): https://learn.mongodb.com
- Zod: https://zod.dev
- OWASP API Security Top 10: https://owasp.org/API-Security

## License

MIT
