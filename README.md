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
POST /register  → hash password → save User → 201
POST /login     → verify password → sign JWT → return token

Protected Route Flow:
  Client Request (Authorization: Bearer <token>)
    → [Auth Middleware] verifies token → req.user set
    → [Validation Middleware] checks input
    → Controller (task routes)
```
