# auth-crud-starter
# FastAPI Backend Assignment – RBAC Project Dashboard

## Overview

This assignment is to build a **FastAPI backend** that implements **CRUD operations with Role-Based Access Control (RBAC)** for a Project Management Dashboard.

The backend must expose a REST API consumed by a React frontend. It handles **authentication via JWT**, enforces **role-based permissions**, and persists data using a relational database.

---

## Objectives

By completing this assignment, you should demonstrate:

- Ability to design and implement REST APIs with FastAPI
- JWT-based authentication and token handling
- Role-based access control using dependency injection
- Clean project structure and separation of concerns
- Database modeling with SQLAlchemy (or Tortoise ORM)
- Basic Python best practices and type annotations

---

## Tech Stack

**Required:**

- Python 3.11+
- FastAPI
- SQLAlchemy (async preferred) + Alembic for migrations
- PostgreSQL or SQLite (SQLite acceptable for local dev)
- `python-jose` or `PyJWT` for JWT
- `passlib[bcrypt]` for password hashing
- Pydantic v2 for schemas

**Optional but encouraged:**

- `fastapi-users` or manual auth implementation
- `httpx` + `pytest` for testing
- Docker + docker-compose
- `.env` support via `python-dotenv` or `pydantic-settings`

---

## User Roles

| Role    | Permissions                                      |
|---------|--------------------------------------------------|
| admin   | Full access: create, read, update, delete any project |
| manager | Create, read, update any project — cannot delete |
| user    | Create projects; read/update only own projects   |

---

## Data Models

### User

| Field        | Type     | Notes                          |
|--------------|----------|--------------------------------|
| id           | UUID/int | Primary key                    |
| email        | string   | Unique, used for login         |
| username     | string   | Display name                   |
| hashed_password | string | bcrypt hashed                 |
| role         | enum     | `admin`, `manager`, `user`     |
| created_at   | datetime | Auto-set on creation           |

### Project

| Field       | Type     | Notes                              |
|-------------|----------|------------------------------------|
| id          | UUID/int | Primary key                        |
| title       | string   | Required, max 255 chars            |
| description | string   | Optional                           |
| status      | enum     | `draft`, `active`, `completed`, `archived` |
| owner_id    | FK       | References `User.id`               |
| created_at  | datetime | Auto-set on creation               |
| updated_at  | datetime | Auto-updated on change             |

---

## API Endpoints

All project routes are prefixed with `/api/v1`.

### Authentication

#### `POST /api/v1/auth/register`

Register a new user.

**Request body:**
```json
{
  "email": "user@example.com",
  "username": "john",
  "password": "securepassword",
  "role": "user"
}
```

**Response `201`:**
```json
{
  "id": 1,
  "email": "user@example.com",
  "username": "john",
  "role": "user",
  "created_at": "2025-01-01T00:00:00Z"
}
```

**Errors:** `422` on validation failure, `400` if email already exists.

---

#### `POST /api/v1/auth/login`

Authenticate and receive a JWT access token.

**Request body (`application/x-www-form-urlencoded`):**
```
username=user@example.com&password=securepassword
```

**Response `200`:**
```json
{
  "access_token": "<jwt>",
  "token_type": "bearer"
}
```

**Errors:** `401` on invalid credentials.

---

#### `GET /api/v1/auth/me`

Return the currently authenticated user's profile.

**Headers:** `Authorization: Bearer <token>`

**Response `200`:**
```json
{
  "id": 1,
  "email": "user@example.com",
  "username": "john",
  "role": "user",
  "created_at": "2025-01-01T00:00:00Z"
}
```

**Errors:** `401` if token is missing or invalid.

---

### Projects

All project routes require a valid JWT (`Authorization: Bearer <token>`).

---

#### `GET /api/v1/projects`

List projects based on the caller's role.

- `admin` / `manager`: returns all projects
- `user`: returns only projects owned by the caller

**Response `200`:**
```json
[
  {
    "id": 1,
    "title": "My Project",
    "description": "A description",
    "status": "draft",
    "owner": {
      "id": 1,
      "username": "john"
    },
    "created_at": "2025-01-01T00:00:00Z",
    "updated_at": "2025-01-01T00:00:00Z"
  }
]
```

---

#### `POST /api/v1/projects`

Create a new project. The owner is set automatically from the JWT.

**Request body:**
```json
{
  "title": "New Project",
  "description": "Optional description"
}
```

**Response `201`:**
```json
{
  "id": 2,
  "title": "New Project",
  "description": "Optional description",
  "status": "draft",
  "owner": {
    "id": 1,
    "username": "john"
  },
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

**Errors:** `422` on missing/invalid fields.

---

#### `GET /api/v1/projects/{id}`

Retrieve a single project by ID.

**Response `200`:** Same shape as the list item above.

**Errors:**
- `404` if project not found
- `403` if `user` role tries to access another user's project

---

#### `PUT /api/v1/projects/{id}`

Update a project's title, description, or status.

**Permission rules:**
- `admin` / `manager`: can update any project
- `user`: can only update own projects

**Request body (all fields optional):**
```json
{
  "title": "Updated Title",
  "description": "Updated description",
  "status": "active"
}
```

**Response `200`:** Updated project object.

**Errors:**
- `403` if the caller lacks permission
- `404` if not found
- `400` on invalid status transition (e.g. `archived` → `draft`)
- `422` on validation failure

---

#### `DELETE /api/v1/projects/{id}`

Delete a project. **Admin only.**

**Response `204`:** No content.

**Errors:**
- `403` if the caller is not `admin`
- `404` if project not found

---

## Error Response Format

All error responses must follow this consistent shape:

```json
{
  "detail": "Human-readable error message"
}
```

For validation errors (`422`), FastAPI returns the default Pydantic error format — this is acceptable.

---

## Status Transition Rules

Valid transitions:

| From       | To                        |
|------------|---------------------------|
| draft      | active                    |
| active     | completed, archived       |
| completed  | archived                  |
| archived   | *(no further transitions)*|

Any other transition must return `400 Bad Request`.

---

## Security Requirements

- Passwords must be hashed with `bcrypt` — never stored in plain text
- JWT tokens must include: `sub` (user id), `role`, and `exp` (expiration)
- Token expiry should be configurable (default: 30 minutes)
- All protected routes must validate the JWT on every request via a FastAPI dependency
- Role checks must be enforced server-side — never trust client-supplied roles

---

## Project Structure (Suggested)

```
app/
├── main.py               # FastAPI app init, router registration
├── config.py             # Settings via pydantic-settings
├── database.py           # DB engine, session, Base
├── models/
│   ├── user.py
│   └── project.py
├── schemas/
│   ├── user.py
│   └── project.py
├── routers/
│   ├── auth.py
│   └── projects.py
├── dependencies/
│   └── auth.py           # get_current_user, require_role
├── services/
│   ├── auth_service.py
│   └── project_service.py
└── alembic/              # DB migrations
```

---

## Seeding

Include a seed script or startup hook that creates:

- At least one user per role (`admin`, `manager`, `user`)
- At least 5 sample projects with varied statuses and owners

This allows the frontend team to immediately test without manual setup.

---

## CORS

Enable CORS for `http://localhost:5173` (default Vite dev server) to allow the frontend to connect during development.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## Submission Guidelines

- Push code to a GitHub repository
- Include a clear commit history
- Add a `README.md` with:
  - How to install dependencies and run the server
  - How to run migrations and seed data
  - A table of seeded test credentials
  - Any assumptions made

---

## Checklist

- [ ] `POST /api/v1/auth/register` — register with role
- [ ] `POST /api/v1/auth/login` — returns JWT
- [ ] `GET /api/v1/auth/me` — returns current user
- [ ] `GET /api/v1/projects` — role-filtered list
- [ ] `POST /api/v1/projects` — create, owner auto-set
- [ ] `GET /api/v1/projects/{id}` — single project with access control
- [ ] `PUT /api/v1/projects/{id}` — update with role checks
- [ ] `DELETE /api/v1/projects/{id}` — admin only
- [ ] Status transition validation
- [ ] Consistent error format
- [ ] Passwords hashed (never plain text)
- [ ] CORS configured for frontend dev server
- [ ] Seed script with one user per role
- [ ] Alembic migrations
- [ ] README with run instructions and test credentials
