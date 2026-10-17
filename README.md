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

