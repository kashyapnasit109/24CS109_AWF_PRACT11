# Practical 7 — Task Management with JWT Authentication

A full-stack Task Management application with JWT-based authentication, bcrypt password hashing, and Express middleware pipeline.

## Tech Stack

- **Backend:** Node.js, Express.js, Mongoose
- **Database:** MongoDB
- **Auth:** bcryptjs, jsonwebtoken
- **Validation:** express-validator
- **Frontend:** HTML, CSS, JavaScript

## Features

- User registration with bcrypt password hashing
- JWT-based login with 1-hour token expiry
- Auth middleware protecting all task routes
- Server-side input validation middleware
- Full CRUD for tasks (scoped to authenticated user)
- `/me` endpoint for current user details
- 401 handling — frontend redirects to login on expired token
- Logout clears token from client

## Setup

```bash
# 1. Clone the repo
git clone https://github.com/24cs73-prince/24CS073_AWDF_P7.git
cd 24CS073_AWDF_P7

# 2. Install dependencies
npm install

# 3. Create .env file (copy from .env.example)
cp .env.example .env
# Edit .env with your MongoDB URI and JWT secret

# 4. Start the server
npm run dev

# 5. Open http://localhost:5000
```

## API Endpoints

### Auth
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/register` | Register a new user |
| POST | `/api/auth/login` | Login and get JWT token |
| GET | `/api/auth/me` | Get current user (protected) |

### Tasks (All protected — require Bearer token)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/tasks` | Get all tasks for logged-in user |
| POST | `/api/tasks` | Create a new task |
| PUT | `/api/tasks/:id` | Update a task |
| DELETE | `/api/tasks/:id` | Delete a task |

## Architecture

```

## Practical 8 — Performance Optimization & Lazy Loading in React

### Overview
Practical 8 introduces route-based code splitting and lazy loading to optimize initial application load time and minimize the initial JavaScript payload size delivered to the user.

### Concepts & Concepts Implemented
- **Lazy Loading**: A performance optimization strategy where resources/components are fetched on-demand when required, rather than downloading the entire application payload upfront.
- **`React.lazy()`**: React standard utility that dynamically imports a component using ECMAScript dynamic `import()`, enabling automated code-splitting by bundlers like Vite.
- **`React.Suspense`**: A React component boundary wrapper that displays fallback UI (`<LoadingPage />`) while pending dynamically-imported route chunks are being fetched across the network.
- **Route-Based Code Splitting**:
  - `Home` (`/`) route remains a static import loaded during initial render.
  - `Projects` (`/projects`) route is lazy-loaded into a separate bundle chunk (`Projects-*.js`).
  - `Contact` (`/contact`) route is lazy-loaded into a separate bundle chunk (`Contact-*.js`).

### Bundle Structure & Loading Mechanics
Route-specific JavaScript is now decoupled from the main bundle and fetched only when navigating to `/projects` or `/contact`. 

Note: Network performance metrics (initial bundle size, load time, JS requests, chunk transfer sizes) are recorded manually via Chrome DevTools under Slow 3G throttling.

## Practical 9 — In-Memory Caching and Query Optimization

### Overview
Practical 9 implements server-side in-memory caching using `node-cache` and database query optimization in MongoDB/Mongoose to reduce response latency and database read load for task retrieval.

### Concepts & Implementation Details
- **In-Memory Caching (`node-cache`)**:
  - Implemented shared cache module in `utils/cache.js` with standard TTL of 60 seconds (`stdTTL: 60`).
  - Read endpoints check memory first before querying MongoDB.
  - Returns `X-Cache: HIT` or `X-Cache: MISS` response headers.
- **Cache Keys & Multi-Tenant Data Isolation**:
  - All tasks list: `tasks:${userId}`
  - Single task item: `task:${userId}:${taskId}`
  - Including the authenticated `userId` guarantees complete user data isolation.
- **Cache Invalidation on Writes**:
  - `POST /api/tasks`: Invalidates `tasks:${userId}` upon successful creation.
  - `PUT /api/tasks/:id`: Invalidates both `tasks:${userId}` and `task:${userId}:${taskId}` upon successful update.
  - `DELETE /api/tasks/:id`: Invalidates both `tasks:${userId}` and `task:${userId}:${taskId}` upon successful deletion.
  - Invalidation only triggers on successful operations (never on validation failure or 404).
- **Query Optimizations**:
  - **Mongoose Compound Index**: `taskSchema.index({ user: 1, createdAt: -1 })` avoids full collection scans (`COLLSCAN`) and memory sort overhead during task list queries.
  - **Single Field Index**: `user: { index: true }` accelerates single-item lookups.
  - **`.lean()` Execution**: Bypasses Mongoose document hydration to return plain JavaScript objects, reducing CPU and RAM overhead.
- **Cache Monitoring & Telemetry**:
  - Protected endpoint `GET /api/tasks/cache-stats` (alias `GET /api/tasks/cache/debug`) reports hit/miss/invalidation counters without exposing secrets or task data.
  - Protected endpoint `POST /api/tasks/cache-stats/reset` resets telemetry counters for clean benchmarking.
- **Cache Bypass for Benchmarking**:
  - Set `CACHE_ENABLED=false` in `.env`, or send request header `x-skip-cache: true` in Thunder Client.

## Practical 10 — Asynchronous Processing with Event-Driven Architecture

### Overview
Practical 10 implements asynchronous background processing within the Task Management application using Node.js's native `EventEmitter` from the built-in `events` module. Decoupling notification side effects from the core HTTP request-response cycle ensures that client write requests return immediately (typically in 5–15 ms) without being blocked by slow downstream tasks such as email delivery, push notifications, or third-party webhooks.

### Architecture & Design
```
Client (Thunder Client / React)
        │
        ▼ POST /api/tasks
┌────────────────────────────────────────────────────────┐
│ Express Router (`routes/tasks.js`)                     │
│ 1. Validate request payload (`validateTask`)           │
│ 2. Save Task to MongoDB (`Task.create`)                │
│ 3. Invalidate User Cache (`cache.invalidateUserTasks`) │
│ 4. Send HTTP 201 Created Response ─────────────────────┼──► Client receives HTTP 201 (~10ms)
│ 5. Schedule event emission via `setImmediate()`        │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼ emit('task-created', taskData)
┌────────────────────────────────────────────────────────┐
│ Shared EventEmitter (`events.js`)                      │
│ Class `TaskEvents extends EventEmitter`                │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼ .on('task-created') listener
┌────────────────────────────────────────────────────────┐
│ Listener Module (`listeners.js`)                       │
│ 1. Log event receipt immediately (Task, User, Time)    │
│ 2. Schedule slow notification via `setTimeout()`       │
│ 3. Asynchronously completes after 2000ms delay         │
│ 4. Log completion timestamp                            │
└────────────────────────────────────────────────────────┘
```

### Components Implemented

1. **Dedicated Event Module (`events.js`)**:
   - Implements `class TaskEvents extends EventEmitter {}`.
   - Exports a single shared instance throughout the application so all routes and listeners communicate via the same event bus.

2. **Dedicated Listener Module (`listeners.js`)**:
   - Registered once during application startup in `server.js`.
   - Listens for `task-created`, logs task title, assigned user ID, and start timestamp, then triggers a simulated 2000 ms notification dispatch using `setTimeout()`.
   - Listens for `task-deleted` (supplementary requirement), logging the deleted task ID, title, user, and deletion timestamp.
   - Registers error listeners (`notification-error` and `error`) to catch asynchronous listener errors and log structured telemetry without crashing the Node.js server process.

3. **Decoupled Task Creation (`routes/tasks.js`)**:
   - Preserves all authentication (`auth.js`), validation (`validateTask`), MongoDB operations (`Task.create`), and Practical 9 cache invalidations (`cache.invalidateUserTasks`).
   - Dispatches the HTTP 201 response immediately before executing background notification work using `setImmediate()`.
   - Real timestamps verify that the HTTP response is sent before the notification handler finishes.

4. **Task Deletion Event Integration (`routes/tasks.js`)**:
   - Preserves ownership checks, database deletion (`Task.findOneAndDelete`), and dual cache invalidation (`tasks:${userId}` and `task:${userId}:${taskId}`).
   - Emits `task-deleted` only upon successful document deletion.
   - Does NOT emit events if the task does not exist (404) or if deletion fails.

### Synchronous vs Asynchronous Handling in Node.js

- **Synchronous Event Listener Invocation**: In Node.js, `EventEmitter.emit()` executes registered event listener functions synchronously in the order they were registered. If a listener executes a synchronous blocking loop (e.g. `while (Date.now() - start < 2000)`), the main thread and HTTP response would freeze.
- **Asynchronous Work Scheduling**: To prevent blocking, our listener receives the event and promptly schedules the heavy notification task onto the Node.js libuv timer phase using `setTimeout(..., 2000)`.
- **Deferred Emission via `setImmediate()`**: In `routes/tasks.js`, `taskEvents.emit()` is wrapped in `setImmediate()`, ensuring the HTTP response is flushed and dispatched to the client before the event handler starts execution.

### In-Process EventEmitter vs Distributed Message Queues

| Feature | Node.js native `EventEmitter` | Distributed Queue (RabbitMQ / BullMQ / Kafka) |
|---|---|---|
| **Infrastructure** | Zero dependencies, built into Node.js runtime | Requires external broker (Redis, RabbitMQ, Kafka cluster) |
| **Durability** | In-memory only; events lost if process restarts | Durable; messages persisted to disk/WAL |
| **Horizontal Scaling** | Isolated to single Node.js process; cannot cross cluster workers | Shared across all worker nodes, clusters, and microservices |
| **Retries & Dead Letter** | Must be coded manually; crashes if unhandled | Built-in ACK/NACK, exponential backoff, and DLQ |
| **Best Used For** | In-process decoupling, local telemetry, small-scale web apps | High-throughput, distributed systems, mission-critical jobs |

### Verification and Test Suite Results

The project includes automated verification test suites for both Practical 9 and Practical 10:

```bash
# Run both verification suites
npm test

# Run individual suites
npm run test:p9
npm run test:p10
```

#### Verification Suite Output:
- **Practical 9 Verification Suite (13 / 13 tests passed)**:
  - Cache hits, misses, TTL, user isolation, cache invalidation on POST, PUT, DELETE, and cache bypass header (`x-skip-cache`).
- **Practical 10 Verification Suite (11 / 11 tests passed)**:
  - Unauthenticated requests rejected (401).
  - Invalid JWT tokens rejected (401).
  - Validation failure returns 400 and suppresses event emission.
  - HTTP 201 returns immediately (~10 ms) while notification completes after 2000 ms.
  - `task-created` payload verified (title, userId, taskId).
  - Practical 9 cache invalidation confirmed on POST (`tasks:${userId}` invalidated).
  - `task-deleted` event emitted on successful deletion.
  - Cache invalidation confirmed on DELETE (both list and single task caches invalidated).
  - Deletion of nonexistent task returns 404 and suppresses event emission.
  - Safe error event handling verified without server crashes.

### Step-by-Step Thunder Client Testing & Screenshots
Refer to [`screenshots/README.md`](file:///c:/Users/jnpat/OneDrive/Desktop/SEM%205/AWDF/Practical_10/screenshots/README.md) for the complete screenshot capture checklist, sample requests, and evidence guidelines.


