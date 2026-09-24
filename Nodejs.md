# Node.js and Express.js Backend Interview Guide

A practical, simple-to-advanced guide for learning Node.js, building APIs with Express, and preparing for backend interviews.

The examples use modern JavaScript with Node.js 20+ and Express 5.

## How to use this guide

1. Read the fundamentals in order.
2. Type the examples instead of only copying them.
3. Build the sample API near the end.
4. Practice the interview questions aloud.
5. Add a database, authentication, tests, and deployment one step at a time.

---

# 1. Backend fundamentals

## What is a backend?

A backend receives requests, applies business rules, reads or writes data, and sends responses.

Typical backend responsibilities:

- HTTP APIs
- Authentication and authorization
- Validation
- Business logic
- Database access
- File and image handling
- Background jobs
- Caching
- Logging and monitoring
- Security
- Deployment and scaling

A common request flow is:

```text
Client -> DNS -> Load balancer -> Web server -> Middleware -> Route -> Controller -> Service -> Database
                                                                                         |
Client <- JSON response <- Error handler <- Controller <---------------------------------+
```

## HTTP basics

An HTTP request contains:

- Method: `GET`, `POST`, `PUT`, `PATCH`, or `DELETE`
- URL and path parameters
- Query parameters
- Headers
- Optional body
- Cookies

An HTTP response contains:

- Status code
- Headers
- Optional body

Important status codes:

| Code | Meaning | Example |
|---|---|---|
| 200 | OK | Successful read or update |
| 201 | Created | New resource created |
| 204 | No Content | Successful delete |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Missing or invalid login |
| 403 | Forbidden | Logged in but not allowed |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Duplicate email |
| 422 | Unprocessable Entity | Well-formed but invalid data |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server failure |
| 503 | Service Unavailable | Dependency or server unavailable |

Use the status code to communicate the result. Do not return `200` for every situation.

## REST API conventions

Example resource: users.

| Action | Method | Endpoint |
|---|---|---|
| List users | GET | `/api/v1/users` |
| Get one user | GET | `/api/v1/users/:id` |
| Create user | POST | `/api/v1/users` |
| Replace user | PUT | `/api/v1/users/:id` |
| Partially update user | PATCH | `/api/v1/users/:id` |
| Delete user | DELETE | `/api/v1/users/:id` |

Good API habits:

- Version public APIs, for example `/api/v1`.
- Use nouns for resources, not verbs: `/users`, not `/getUsers`.
- Validate every external input.
- Return a predictable response shape.
- Paginate collections.
- Never expose passwords, tokens, or internal stack traces.
- Document authentication, errors, pagination, and examples.

Example response shape:

```json
{
  "success": true,
  "data": {
    "id": "usr_123",
    "name": "Asha"
  },
  "requestId": "req_456"
}
```

Example error:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request is invalid",
    "details": [
      { "field": "email", "message": "Email is required" }
    ]
  },
  "requestId": "req_456"
}
```

---

# 2. JavaScript required for Node.js

## Variables and functions

Prefer `const`; use `let` only when a value changes. Avoid `var` in modern code.

```js
const port = 3000;
let requestCount = 0;

function add(a, b) {
  return a + b;
}

const multiply = (a, b) => a * b;
```

## Destructuring and spread

```js
const user = { id: 1, name: 'Asha', role: 'admin' };
const { id, name } = user;
const safeUser = { id, name };

const numbers = [1, 2, 3];
const moreNumbers = [...numbers, 4];
```

## Error handling

```js
try {
  const parsed = JSON.parse('{"ok":true}');
  console.log(parsed.ok);
} catch (error) {
  console.error('Could not parse JSON', error.message);
}
```

Do not silently swallow errors. Either handle them or pass them to the central error handler.

## Promises and async/await

A Promise represents a future result: pending, fulfilled, or rejected.

```js
function wait(milliseconds) {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

async function run() {
  await wait(100);
  return 'done';
}
```

Sequential work:

```js
const user = await getUser(id);
const orders = await getOrders(user.id);
```

Independent work should run in parallel:

```js
const [user, settings] = await Promise.all([
  getUser(id),
  getSettings(id)
]);
```

## Event loop in simple terms

Node.js runs JavaScript on an event loop. It can handle many I/O operations without creating one thread per request.

```js
console.log('1');
setTimeout(() => console.log('3'), 0);
Promise.resolve().then(() => console.log('2'));
console.log('4');
```

Typical output:

```text
1
4
2
3
```

Microtasks such as Promise callbacks run before timer callbacks after the current synchronous work finishes.

Important interview point: Node.js is not magically non-blocking. CPU-heavy synchronous code blocks the event loop and delays every request on that process.

Bad:

```js
import crypto from 'node:crypto';

app.get('/bad', (req, res) => {
  crypto.pbkdf2Sync('secret', 'salt', 500000, 64, 'sha512');
  res.json({ ok: true });
});
```

Better: use the asynchronous API, a worker thread, or a separate job service for CPU-heavy work.

## Common JavaScript mistakes

- Forgetting `await`.
- Using `forEach(async () => {})` and expecting it to wait.
- Mutating shared objects unexpectedly.
- Comparing values with `==` instead of `===`.
- Assuming a rejected Promise is caught automatically.
- Blocking the event loop with large loops, synchronous file APIs, or expensive JSON work.
- Trusting client-provided object fields such as `role`, `userId`, or `price`.

Correct async iteration:

```js
for (const item of items) {
  await processItem(item);
}
```

Or parallel processing when safe:

```js
await Promise.all(items.map((item) => processItem(item)));
```

---

# 3. Node.js from zero

## What is Node.js?

Node.js is a JavaScript runtime built on the V8 engine. It provides server-side APIs for networking, files, streams, processes, cryptography, and more.

Node.js is a good fit for I/O-heavy services such as APIs, real-time applications, gateways, and streaming systems. It needs special care for CPU-heavy workloads.

## Install and run a project

```bash
mkdir backend-api
cd backend-api
npm init -y
npm install express
npm install --save-dev nodemon
```

`package.json` scripts:

```json
{
  "type": "module",
  "scripts": {
    "dev": "node --watch src/server.js",
    "start": "node src/server.js"
  }
}
```

Use `.env` for local configuration, but never commit secrets:

```env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://user:password@localhost:5432/app
JWT_SECRET=replace-this-locally
```

Read configuration at startup and fail fast for required values:

```js
const port = Number(process.env.PORT || 3000);

if (!process.env.JWT_SECRET && process.env.NODE_ENV === 'production') {
  throw new Error('JWT_SECRET is required in production');
}
```

## Built-in HTTP server

Express is useful, but first understand the Node.js primitive:

```js
import http from 'node:http';

const server = http.createServer((request, response) => {
  response.writeHead(200, { 'content-type': 'application/json' });
  response.end(JSON.stringify({ message: 'Hello from Node.js' }));
});

server.listen(3000, () => {
  console.log('Listening on http://localhost:3000');
});
```

## Modules

Modern projects commonly use ES modules:

```js
// math.js
export function add(a, b) {
  return a + b;
}

// app.js
import { add } from './math.js';
console.log(add(2, 3));
```

CommonJS is also found in older projects:

```js
const express = require('express');
module.exports = router;
```

Do not mix module systems casually. Follow the project configuration.

## Useful Node.js modules

| Module | Use |
|---|---|
| `node:fs/promises` | Async file operations |
| `node:path` | Safe path construction |
| `node:http` | HTTP servers |
| `node:crypto` | Hashing, random values, cryptography |
| `node:stream` | Large data processing |
| `node:events` | Event emitters |
| `node:worker_threads` | CPU work in worker threads |
| `node:child_process` | Run external processes carefully |
| `node:test` | Built-in test runner |

## File example

```js
import { readFile, writeFile } from 'node:fs/promises';

const filePath = new URL('./data.json', import.meta.url);
const data = JSON.parse(await readFile(filePath, 'utf8'));
data.updatedAt = new Date().toISOString();
await writeFile(filePath, JSON.stringify(data, null, 2));
```

Do not build file paths directly from untrusted input. Prevent path traversal with validation and safe path resolution.

## Streams and backpressure

Streams process data in chunks instead of loading the entire payload into memory.

```js
import { createReadStream } from 'node:fs';
import { createServer } from 'node:http';

createServer((request, response) => {
  response.writeHead(200, { 'content-type': 'text/plain' });
  createReadStream('./large.log').pipe(response);
}).listen(3000);
```

Backpressure means the producer slows down when the consumer cannot keep up. `.pipe()` helps coordinate this flow.

---

# 4. Express.js fundamentals

## What is Express?

Express is a minimal web framework for Node.js. It provides routing, middleware composition, request and response helpers, and a large ecosystem.

## Small Express application

```js
import express from 'express';

const app = express();
app.use(express.json());

app.get('/health', (request, response) => {
  response.json({ status: 'ok' });
});

app.listen(3000, () => {
  console.log('API listening on port 3000');
});
```

Keep application creation separate from starting the server. It makes testing easier.

```js
// src/app.js
import express from 'express';

export function createApp() {
  const app = express();
  app.use(express.json({ limit: '1mb' }));
  app.get('/health', (request, response) => {
    response.json({ status: 'ok' });
  });
  return app;
}

// src/server.js
import { createApp } from './app.js';

const app = createApp();
const server = app.listen(process.env.PORT || 3000);

function shutdown(signal) {
  console.log(`${signal} received`);
  server.close(() => process.exit(0));
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

## Middleware

Middleware receives `request`, `response`, and `next`. It can read or change the request, send a response, or pass control onward.

```js
function requestTimer(request, response, next) {
  const startedAt = Date.now();

  response.on('finish', () => {
    console.log(request.method, request.originalUrl, Date.now() - startedAt, 'ms');
  });

  next();
}

app.use(requestTimer);
```

Order matters:

```js
app.use(express.json());
app.use(requestLogger);
app.use('/api/v1/users', userRoutes);
app.use(notFoundHandler);
app.use(errorHandler);
```

A middleware that does not call `next()` and does not send a response will leave the request hanging.

## Routing

```js
import { Router } from 'express';

const router = Router();

router.get('/', listUsers);
router.get('/:id', getUser);
router.post('/', createUser);
router.patch('/:id', updateUser);
router.delete('/:id', deleteUser);

app.use('/api/v1/users', router);
```

Request data:

```js
app.get('/search/:category', (request, response) => {
  const category = request.params.category;
  const page = Number(request.query.page || 1);
  const authorization = request.get('authorization');
  response.json({ category, page, authorizationPresent: Boolean(authorization) });
});
```

## Express 5 async errors

Express 5 forwards rejected Promises from route handlers to the error middleware. A central error handler is still required.

```js
app.get('/users/:id', async (request, response) => {
  const user = await userService.findById(request.params.id);
  response.json({ data: user });
});
```

If a project uses Express 4, use an async wrapper or explicitly call `next(error)`.

## Error handling

Error middleware has four parameters and must be registered after routes:

```js
function notFoundHandler(request, response) {
  response.status(404).json({
    success: false,
    error: { code: 'NOT_FOUND', message: 'Route not found' }
  });
}

function errorHandler(error, request, response, next) {
  if (response.headersSent) {
    return next(error);
  }

  console.error(error);
  const status = error.statusCode || 500;
  const message = status >= 500 ? 'Internal server error' : error.message;

  response.status(status).json({
    success: false,
    error: { code: error.code || 'INTERNAL_ERROR', message }
  });
}
```

Create operational errors instead of throwing random strings:

```js
export class AppError extends Error {
  constructor(message, statusCode, code = 'APP_ERROR') {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
  }
}
```

Never return stack traces or database errors to clients in production.

---

# 5. A maintainable backend structure

A practical feature-oriented structure:

```text
src/
  app.js
  server.js
  config/
    env.js
  routes/
    user.routes.js
  controllers/
    user.controller.js
  services/
    user.service.js
  repositories/
    user.repository.js
  middleware/
    auth.js
    error.js
    validate.js
  schemas/
    user.schema.js
  db/
    client.js
  utils/
    errors.js
    logger.js
  tests/
    user.test.js
```

Responsibilities:

- Route: maps HTTP method and path to a controller.
- Middleware: cross-cutting request behavior such as auth or validation.
- Controller: translates HTTP input to a service call and formats the response.
- Service: business rules and orchestration.
- Repository: database queries and persistence details.
- Schema: input and output validation.
- Configuration: environment parsing and startup settings.

Controller example:

```js
export async function getUser(request, response) {
  const user = await userService.findById(request.params.id);
  response.json({ success: true, data: user });
}
```

Service example:

```js
export async function findById(id) {
  const user = await userRepository.findById(id);
  if (!user) {
    throw new AppError('User not found', 404, 'USER_NOT_FOUND');
  }
  return user;
}
```

Do not place SQL, password hashing, business rules, and response formatting in one route file.

---

# 6. Validation and safe input handling

Validate:

- Body
- Parameters
- Query strings
- Headers where relevant
- File type and size
- Cross-field rules

Example with Zod:

```bash
npm install zod
```

```js
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string().trim().min(2).max(100),
  email: z.string().trim().toLowerCase().email(),
  age: z.number().int().min(13).max(120).optional()
}).strict();

export function validateBody(schema) {
  return (request, response, next) => {
    const result = schema.safeParse(request.body);
    if (!result.success) {
      return response.status(422).json({
        success: false,
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid request body',
          details: result.error.issues
        }
      });
    }
    request.body = result.data;
    next();
  };
}
```

Never trust:

```js
// Dangerous: the client controls fields it should not control.
await User.create(request.body);
```

Prefer an allowlist:

```js
const { name, email } = request.body;
await User.create({ name, email });
```

---

# 7. Databases and persistence

## SQL versus NoSQL

SQL databases such as PostgreSQL and MySQL are strong choices for relational data, constraints, joins, transactions, and reporting.

Document databases such as MongoDB are useful when data is naturally document-shaped and access patterns are understood.

The correct choice depends on consistency, relationships, query patterns, scale, team experience, and operational requirements. Do not choose only because a technology is popular.

## Connection pooling

A pool reuses database connections. Creating a new connection for every request is slow and can exhaust the database.

```js
import pg from 'pg';

const { Pool } = pg;
export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30000
});
```

## Parameterized queries

Never concatenate user input into SQL.

```js
const result = await pool.query(
  'SELECT id, name, email FROM users WHERE id = $1',
  [request.params.id]
);
```

This helps prevent SQL injection.

## Transactions

Use a transaction when multiple writes must succeed or fail together.

```js
const client = await pool.connect();
try {
  await client.query('BEGIN');
  const order = await client.query(
    'INSERT INTO orders(user_id, total) VALUES ($1, $2) RETURNING id',
    [userId, total]
  );
  await client.query(
    'INSERT INTO payments(order_id, amount) VALUES ($1, $2)',
    [order.rows[0].id, total]
  );
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
```

## Database essentials for interviews

- Primary key: uniquely identifies a row.
- Foreign key: enforces a relationship.
- Index: speeds reads but adds storage and write cost.
- Composite index: useful for queries filtering by multiple columns, with column order matters.
- Unique constraint: enforces uniqueness at the database boundary.
- Migration: versioned schema change.
- Transaction: atomic unit of work.
- Isolation: controls how concurrent transactions see changes.
- N+1 query problem: one query for a list followed by one query per item.
- Soft delete: mark deleted records instead of physically removing them.

Put correctness constraints in the database as well as in application validation.

---

# 8. Authentication and authorization

## Authentication versus authorization

- Authentication: Who are you?
- Authorization: What are you allowed to do?

## Password storage

Never store plain-text passwords. Use a slow password hashing algorithm such as Argon2id or bcrypt with an appropriate cost.

```bash
npm install argon2
```

```js
import argon2 from 'argon2';

const passwordHash = await argon2.hash(password);
const isValid = await argon2.verify(passwordHash, password);
```

## Session authentication

A session system usually stores a random session identifier in an `HttpOnly`, `Secure`, `SameSite` cookie. The server stores session data in a shared store such as Redis or a database.

Advantages:

- Easy revocation.
- Small cookie.
- Good fit for browser applications.

## JWT authentication

A JWT contains claims and a signature. It is usually sent as a Bearer token.

```text
Authorization: Bearer <token>
```

JWT middleware concept:

```js
import jwt from 'jsonwebtoken';

export function requireAuth(request, response, next) {
  const header = request.get('authorization');
  const token = header?.startsWith('Bearer ')
    ? header.slice(7)
    : null;

  if (!token) {
    return response.status(401).json({ message: 'Authentication required' });
  }

  try {
    request.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    response.status(401).json({ message: 'Invalid or expired token' });
  }
}
```

JWT cautions:

- Keep access tokens short-lived.
- Protect refresh tokens carefully.
- Have a revocation or rotation strategy.
- Do not put secrets in JWT payloads; payloads are readable.
- Validate issuer, audience, algorithm, and expiration.
- Do not accept an algorithm selected by an untrusted token.

Role authorization:

```js
export function requireRole(...allowedRoles) {
  return (request, response, next) => {
    if (!allowedRoles.includes(request.user.role)) {
      return response.status(403).json({ message: 'Forbidden' });
    }
    next();
  };
}

app.delete(
  '/api/v1/users/:id',
  requireAuth,
  requireRole('admin'),
  deleteUser
);
```

---

# 9. Backend security checklist

## Essential protections

```bash
npm install helmet cors express-rate-limit
```

```js
import cors from 'cors';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

app.use(helmet());
app.use(cors({
  origin: ['https://app.example.com'],
  credentials: true
}));
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 300,
  standardHeaders: 'draft-8',
  legacyHeaders: false
}));
```

Security checklist:

- Use HTTPS in production.
- Set security headers with Helmet.
- Configure CORS with an allowlist, not `*` with credentials.
- Limit body size and upload size.
- Rate-limit login, password reset, and expensive endpoints.
- Validate and normalize all external data.
- Use parameterized queries or a safe ORM.
- Prevent mass assignment with field allowlists.
- Use secure, HttpOnly, SameSite cookies.
- Protect cookie-based state-changing requests from CSRF.
- Avoid leaking whether an email exists during account recovery.
- Redact tokens, passwords, and personal data from logs.
- Keep dependencies updated and run `npm audit` as one input, not as the entire security program.
- Use dependency and secret scanning in CI.
- Set timeouts on outbound requests.
- Do not use `eval` with untrusted data.
- Validate redirect URLs to avoid open redirects.
- Set a Content Security Policy where applicable.

### CSRF

CSRF matters mainly when browsers automatically attach authentication cookies. A malicious site can cause a victim's browser to send a request. Common defenses are SameSite cookies, CSRF tokens, and checking the `Origin` header.

Bearer tokens kept outside cookies avoid automatic cookie attachment but introduce token storage and XSS considerations.

### XSS

Escape user content when rendering HTML. For JSON APIs, do not assume JSON automatically makes every downstream consumer safe. Validate content and use browser security headers in web applications.

---

# 10. API design in production

## Pagination

Offset pagination is simple:

```text
GET /orders?page=2&limit=20
```

Cursor pagination is usually more stable for changing large datasets:

```text
GET /orders?limit=20&after=eyJpZCI6MTAwfQ
```

Always:

- Set a maximum limit.
- Use stable ordering.
- Return pagination metadata or a next cursor.
- Avoid exposing database implementation details unnecessarily.

## Filtering and sorting

Allowlist fields and directions:

```js
const allowedSortFields = new Set(['createdAt', 'name']);
const sort = allowedSortFields.has(request.query.sort)
  ? request.query.sort
  : 'createdAt';
const direction = request.query.direction === 'asc' ? 'asc' : 'desc';
```

Never insert an arbitrary query parameter directly into SQL or a database command.

## Idempotency

An idempotent operation can be retried without creating an unintended second effect. `GET`, `PUT`, and `DELETE` are intended to be idempotent; `POST` is not necessarily idempotent.

For payments or order creation, accept an idempotency key and store the result:

```text
Idempotency-Key: checkout-8f2b
```

A retry with the same key should return the original result rather than create a second order.

## API versioning

Common strategies:

- URL: `/api/v1/users`
- Header: `Accept: application/vnd.example.v1+json`
- Query parameter: `/users?version=1`

URL versioning is easiest to understand. Keep old versions during a migration window and document deprecations.

## Caching

Use HTTP caching headers where possible:

```js
response.set('Cache-Control', 'public, max-age=60');
```

Redis is useful for shared application caches, sessions, rate limits, and short-lived locks. Define invalidation rules before adding a cache.

---

# 11. Files, queues, and real-time work

## File uploads

For uploads:

- Limit size.
- Validate MIME type and file signature.
- Generate your own storage name.
- Store outside the application process when possible.
- Scan files where required.
- Never trust the original filename.
- Avoid serving uploaded files with executable content types.

Use object storage such as S3-compatible storage for production-scale files. Generate short-lived signed upload URLs when clients can upload directly.

## Background jobs

Do not keep an HTTP request open for slow, retryable work such as emails, report generation, or image processing.

```text
HTTP request -> Store job -> Queue -> Worker -> External service
                         <- status endpoint or notification
```

A worker should support:

- Retries with backoff.
- Dead-letter handling.
- Idempotent job processing.
- Visibility or lease timeouts.
- Metrics and failure alerts.

## WebSockets and Server-Sent Events

WebSockets provide two-way communication. Server-Sent Events provide one-way server-to-client streaming over HTTP. Both need connection limits, authentication, heartbeat handling, reconnect behavior, and horizontal scaling strategy.

---

# 12. Testing

## Test levels

- Unit test: one function or module in isolation.
- Integration test: multiple real modules, often with a test database.
- API test: sends HTTP requests to the app.
- End-to-end test: tests a user workflow through deployed-like systems.
- Load test: measures behavior under traffic.

Aim for valuable coverage, not a percentage target alone.

## Node test example

```js
import test from 'node:test';
import assert from 'node:assert/strict';

function isAdult(age) {
  return age >= 18;
}

test('isAdult returns true for adults', () => {
  assert.equal(isAdult(18), true);
  assert.equal(isAdult(17), false);
});
```

## API testing with Supertest

```bash
npm install --save-dev supertest
```

```js
import test from 'node:test';
import assert from 'node:assert/strict';
import request from 'supertest';
import { createApp } from '../app.js';

test('GET /health returns healthy status', async () => {
  const response = await request(createApp()).get('/health');
  assert.equal(response.status, 200);
  assert.equal(response.body.status, 'ok');
});
```

Test success and failure paths:

- Valid request.
- Missing required field.
- Invalid identifier.
- Not found.
- Unauthorized.
- Forbidden.
- Duplicate resource.
- Dependency failure.
- Rate limit behavior.

Do not make tests depend on production services. Use test databases, containers, mocks, or fakes deliberately.

---

# 13. Logging, monitoring, and reliability

## Structured logging

Use JSON logs in production:

```js
console.log(JSON.stringify({
  level: 'info',
  event: 'request_completed',
  method: request.method,
  path: request.originalUrl,
  statusCode: response.statusCode,
  durationMs: Date.now() - startedAt,
  requestId: request.id
}));
```

Log:

- Request ID.
- Route and status.
- Duration.
- Error code and stack internally.
- Important business events.

Do not log passwords, access tokens, full payment data, or unnecessary personal data.

## Health endpoints

- Liveness: process is running.
- Readiness: process can receive traffic and required dependencies are available.

```js
app.get('/live', (request, response) => {
  response.json({ status: 'ok' });
});

app.get('/ready', async (request, response) => {
  await database.query('SELECT 1');
  response.json({ status: 'ready' });
});
```

A readiness check should fail when the instance cannot serve useful traffic. Keep checks bounded by timeouts.

## Metrics

Track:

- Request count.
- Error rate.
- Latency percentiles such as p50, p95, and p99.
- Event-loop lag.
- CPU and memory.
- Database pool usage.
- Queue depth and job failures.
- External dependency latency.

## Graceful shutdown

On `SIGTERM`:

1. Stop accepting new requests.
2. Allow in-flight requests to finish for a bounded period.
3. Close database and queue connections.
4. Exit.

Always include a forced timeout so a broken connection cannot keep a deployment stuck forever.

---

# 14. Performance and scaling

## Common performance problems

- Blocking the event loop.
- Missing database indexes.
- N+1 queries.
- Returning too much data.
- No pagination.
- Repeated external calls without caching.
- Creating database connections per request.
- Memory leaks from global arrays or event listeners.
- Unbounded queues or request bodies.
- Logging large payloads.

## Horizontal scaling

Run multiple stateless Node.js instances behind a load balancer. Store shared state in a database, Redis, or another shared system, not process memory.

Example:

```text
Client -> Load balancer -> Node instance 1
                       -> Node instance 2
                       -> Node instance 3
                         |
                         +-> PostgreSQL / Redis / Queue
```

If using WebSockets, configure sticky sessions or a shared pub/sub adapter as appropriate.

## Worker threads and processes

- Worker threads: CPU-heavy JavaScript in another thread.
- Child processes: isolated external process.
- Cluster or multiple processes: use all CPU cores, usually managed by a process manager or container platform.
- Separate service: isolate a workload with different scaling or failure characteristics.

Do not add microservices merely to create more deployment units. Start with a modular monolith unless independent scaling, ownership, or isolation justifies a split.

---

# 15. Deployment and DevOps

## Production checklist

- Pin and review dependency versions.
- Set production environment variables through a secret manager.
- Run migrations safely.
- Build a small container image.
- Run as a non-root user.
- Use HTTPS at the edge.
- Configure health and readiness checks.
- Set memory and CPU limits.
- Capture structured logs.
- Configure alerts.
- Back up databases and test restoring them.
- Define rollback steps.
- Use graceful shutdown.
- Verify timeouts and retry policies.

## Example Dockerfile

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY src ./src
USER node
EXPOSE 3000
CMD ["node", "src/server.js"]
```

Do not bake secrets into the image.

## CI pipeline stages

```text
Install -> Lint -> Type check -> Unit tests -> Integration tests -> Build image -> Scan -> Deploy
```

Use rolling, blue-green, or canary deployments depending on risk. A database migration must be compatible with both the old and new application versions during a rolling deployment.

---

# 16. Complete small Express API example

This example is intentionally in-memory so it runs without a database. It demonstrates routing, validation, authentication-shaped middleware, errors, pagination, and graceful shutdown. Replace the store with a repository backed by PostgreSQL or another database in production.

## Install

```bash
mkdir express-api-example
cd express-api-example
npm init -y
npm install express helmet cors zod
```

Set `"type": "module"` in `package.json` and create `src/server.js`:

```js
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { randomUUID } from 'node:crypto';
import { z } from 'zod';

const app = express();
const port = Number(process.env.PORT || 3000);
const users = new Map();

app.use(helmet());
app.use(cors({ origin: 'http://localhost:5173' }));
app.use(express.json({ limit: '1mb' }));

app.use((request, response, next) => {
  request.requestId = randomUUID();
  response.set('X-Request-Id', request.requestId);
  next();
});

const createUserSchema = z.object({
  name: z.string().trim().min(2).max(100),
  email: z.string().trim().toLowerCase().email()
}).strict();

function validate(schema) {
  return (request, response, next) => {
    const result = schema.safeParse(request.body);
    if (!result.success) {
      return next(new HttpError(422, 'VALIDATION_ERROR', 'Invalid request body', result.error.issues));
    }
    request.body = result.data;
    next();
  };
}

class HttpError extends Error {
  constructor(statusCode, code, message, details) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.details = details;
  }
}

function requireApiKey(request, response, next) {
  if (request.get('x-api-key') !== process.env.API_KEY) {
    return next(new HttpError(401, 'UNAUTHORIZED', 'A valid API key is required'));
  }
  next();
}

app.get('/health', (request, response) => {
  response.json({ success: true, data: { status: 'ok' } });
});

app.get('/api/v1/users', (request, response) => {
  const page = Math.max(Number(request.query.page) || 1, 1);
  const limit = Math.min(Math.max(Number(request.query.limit) || 10, 1), 50);
  const allUsers = [...users.values()];
  const start = (page - 1) * limit;
  const data = allUsers.slice(start, start + limit);

  response.json({
    success: true,
    data,
    pagination: {
      page,
      limit,
      total: allUsers.length,
      totalPages: Math.ceil(allUsers.length / limit)
    }
  });
});

app.get('/api/v1/users/:id', (request, response, next) => {
  const user = users.get(request.params.id);
  if (!user) {
    return next(new HttpError(404, 'USER_NOT_FOUND', 'User not found'));
  }
  response.json({ success: true, data: user });
});

app.post('/api/v1/users', requireApiKey, validate(createUserSchema), (request, response, next) => {
  const duplicate = [...users.values()].some((user) => user.email === request.body.email);
  if (duplicate) {
    return next(new HttpError(409, 'EMAIL_EXISTS', 'Email is already registered'));
  }

  const user = {
    id: randomUUID(),
    ...request.body,
    createdAt: new Date().toISOString()
  };
  users.set(user.id, user);
  response.status(201).json({ success: true, data: user });
});

app.delete('/api/v1/users/:id', requireApiKey, (request, response, next) => {
  if (!users.delete(request.params.id)) {
    return next(new HttpError(404, 'USER_NOT_FOUND', 'User not found'));
  }
  response.status(204).send();
});

app.use((request, response, next) => {
  next(new HttpError(404, 'ROUTE_NOT_FOUND', 'Route not found'));
});

app.use((error, request, response, next) => {
  if (response.headersSent) {
    return next(error);
  }

  const statusCode = error.statusCode || 500;
  console.error({
    requestId: request.requestId,
    error: error.message,
    stack: error.stack
  });

  response.status(statusCode).json({
    success: false,
    error: {
      code: error.code || 'INTERNAL_ERROR',
      message: statusCode >= 500 ? 'Internal server error' : error.message,
      ...(error.details ? { details: error.details } : {})
    },
    requestId: request.requestId
  });
});

const server = app.listen(port, () => {
  console.log(`API listening on http://localhost:${port}`);
});

function shutdown(signal) {
  console.log(`${signal} received`);
  server.close(() => process.exit(0));
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

Run it:

```bash
API_KEY=local-secret node src/server.js
```

Try it:

```bash
curl http://localhost:3000/health
curl http://localhost:3000/api/v1/users
curl -X POST http://localhost:3000/api/v1/users \
  -H 'content-type: application/json' \
  -H 'x-api-key: local-secret' \
  -d '{"name":"Asha","email":"asha@example.com"}'
```

For production, replace the in-memory `Map` with a repository, use a real authentication system, configure CORS from trusted origins, add rate limits, add tests, and use a managed database.

---

# 17. System design interview framework

When asked to design a backend, follow this sequence:

1. Clarify users, use cases, and non-functional requirements.
2. Estimate traffic, storage, payload sizes, and growth.
3. Define API endpoints and important data models.
4. Choose database and explain why.
5. Draw the request flow.
6. Discuss caching, queues, and external services.
7. Explain consistency and failure handling.
8. Explain security and authorization.
9. Explain observability and deployment.
10. Identify bottlenecks and future scaling options.

Questions to ask:

- Read-heavy or write-heavy?
- Strong consistency or eventual consistency?
- Maximum acceptable latency?
- Peak traffic or average traffic?
- Data retention requirements?
- Need for audit history?
- What happens when a dependency is down?
- Can requests be retried safely?
- What must be private?
- What is the recovery point and recovery time objective?

## Example: URL shortener components

```text
Client -> API -> URL service -> Database
                       |             |
                       +-> Cache <---+
                       +-> Analytics queue -> Worker -> Analytics store
```

Important decisions:

- Generate unique short codes.
- Put a unique constraint on the code.
- Cache popular redirects.
- Apply abuse and rate limits.
- Decide whether analytics are synchronous or asynchronous.
- Define expiration and deletion behavior.
- Protect administrative endpoints.

---

# 18. Interview questions and concise answers

## Node.js

### What is the event loop?

It is the mechanism that lets Node.js coordinate synchronous JavaScript with asynchronous I/O callbacks. It does not make CPU-heavy JavaScript non-blocking.

### Is Node.js single-threaded?

JavaScript execution in a Node.js process is primarily single-threaded, but Node.js and its libraries use the operating system and a libuv thread pool for some I/O. Worker threads can run JavaScript CPU work separately.

### What blocks the event loop?

Large synchronous loops, synchronous filesystem calls, expensive parsing or serialization, regular expressions with catastrophic backtracking, and CPU-heavy cryptography.

### What is a stream?

A stream processes data incrementally. It reduces memory usage and can start producing output before all input is available.

### `process.nextTick` versus `setImmediate`?

`process.nextTick` runs very soon after the current operation and can starve I/O if abused. `setImmediate` runs in a later event-loop phase, commonly after I/O callbacks.

### How do you handle uncaught errors?

Prevent them with validation and central error handling, log enough context, shut down safely when process state is unknown, and let a supervisor restart the process. Do not continue blindly after an unrecoverable initialization failure.

## Express

### What is middleware?

A function that can inspect or modify a request and response, end the request, or pass control to the next middleware.

### Why does middleware order matter?

Express executes middleware in registration order. Parsers must run before code reads the body, auth before protected routes, and the error handler after all routes.

### How do you handle async errors?

Use Express 5 Promise forwarding or an explicit async wrapper for Express 4, then send errors to one central error handler.

### How do you structure an Express application?

Keep routes thin and separate routing, middleware, controllers, services, repositories, validation, configuration, and error handling.

### How do you secure an Express API?

Validate input, use Helmet, restrict CORS, rate-limit sensitive routes, use HTTPS, protect cookies, parameterize queries, hash passwords, redact logs, limit body sizes, and keep dependencies updated.

## Databases

### Why use indexes?

Indexes reduce read work for matching queries, but consume storage and can slow writes. Index the access patterns that matter and verify with query plans.

### What is a transaction?

A group of operations that follows atomicity, consistency, isolation, and durability. It prevents partial updates when operations must succeed together.

### What is the N+1 problem?

Fetching a list with one query and then making one query per item. Fix it with joins, batching, eager loading, or a carefully designed aggregate query.

### When would you use Redis?

For shared cache, sessions, rate limits, distributed locks, pub/sub, or short-lived data. Do not treat it as a permanent source of truth unless the design explicitly supports that.

## Security

### Authentication versus authorization?

Authentication identifies a principal. Authorization checks whether that principal may perform an action on a resource.

### How do you prevent SQL injection?

Use parameterized queries or a safe query builder and never concatenate untrusted input into SQL.

### What is CSRF?

A browser is tricked into sending an authenticated state-changing request. Defend with SameSite cookies, CSRF tokens, and origin checks when cookie authentication is used.

### How do you store passwords?

Use Argon2id or bcrypt with a suitable work factor, unique salts, and a safe password reset flow. Never encrypt or hash passwords with fast general-purpose hashes such as plain SHA-256.

## System design and operations

### How do you scale an Express API?

Keep instances stateless, run multiple instances behind a load balancer, use database pooling, cache carefully, move slow work to queues, add indexes, and measure before optimizing.

### What is graceful shutdown?

Stop accepting new traffic, finish in-flight work within a deadline, close connections, and exit so the supervisor can safely replace the process.

### What should be monitored?

Error rate, latency percentiles, throughput, event-loop lag, memory, CPU, database pool usage, queue depth, dependency failures, and business-level success metrics.

### How do you make an API reliable?

Use timeouts, bounded retries with jitter, circuit breaking where useful, idempotency keys, durable queues, health checks, backups, tested recovery, structured logs, and clear ownership.

---

# 19. Final backend checklist

Before calling an API production-ready, verify:

- [ ] Requirements and API contract are documented.
- [ ] Every input is validated and normalized.
- [ ] Authentication and authorization are explicit.
- [ ] Passwords and secrets are protected.
- [ ] Database queries are parameterized.
- [ ] Important invariants have database constraints.
- [ ] Errors have stable codes and safe messages.
- [ ] Logs include request IDs and redact secrets.
- [ ] Health, readiness, metrics, and alerts exist.
- [ ] Timeouts and bounded retries exist for dependencies.
- [ ] Slow work uses a queue or worker.
- [ ] Pagination and maximum limits exist.
- [ ] Rate limits and body limits exist.
- [ ] Tests cover success and failure paths.
- [ ] CI runs tests and security checks.
- [ ] Deployment supports rollback and graceful shutdown.
- [ ] Backups have been restored in a test.
- [ ] Load and failure behavior are understood.

The strongest interview answers connect code to tradeoffs: correctness, security, latency, cost, operability, and failure behavior.
