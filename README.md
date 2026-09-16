# Expense Tracker API

A REST API for tracking personal expenses, built with Node.js, Express, Sequelize,
and SQLite. Each user authenticates with a JWT and can only see/manage their own
expenses.

## Tech stack

- **Runtime:** Node.js + Express
- **Database:** SQLite (via Sequelize ORM) — swap the `dialect` in `src/config/db.js`
  for Postgres/MySQL/etc. with minimal changes if you want a production DB.
- **Auth:** JSON Web Tokens (`jsonwebtoken`) + password hashing (`bcryptjs`)
- **Validation:** `express-validator`

## Getting started

```bash
# 1. Install dependencies
npm install

# 2. Configure environment
cp .env.example .env
# then edit .env and set a strong JWT_SECRET

# 3. Run the server (creates database.sqlite automatically on first run)
npm start
# or, for auto-restart during development:
npm run dev
```

The API will be available at `http://localhost:3000`. Health check: `GET /health`.

## Data model

**User**: `id, name, email (unique), password (hashed), createdAt, updatedAt`

**Expense**: `id, title, amount, category, description, date, userId, createdAt, updatedAt`

Allowed categories: `Groceries, Leisure, Electronics, Utilities, Clothing, Health, Others`

## Authentication

Every `/api/expenses/*` route requires a valid JWT in the `Authorization` header:

```
Authorization: Bearer <token>
```

Get a token from `POST /api/auth/signup` or `POST /api/auth/login`.

---

## API Reference

### `POST /api/auth/signup`
Create a new user account.

Request body:
```json
{ "name": "Jane Doe", "email": "jane@example.com", "password": "secret123" }
```
Response `201`:
```json
{ "user": { "id": 1, "name": "Jane Doe", "email": "jane@example.com" }, "token": "<jwt>" }
```

### `POST /api/auth/login`
Log in with email/password.

Request body:
```json
{ "email": "jane@example.com", "password": "secret123" }
```
Response `200`: same shape as signup.

---

### `POST /api/expenses`
Create a new expense. **Auth required.**

Request body:
```json
{
  "title": "Weekly groceries",
  "amount": 54.20,
  "category": "Groceries",
  "description": "Whole Foods run",
  "date": "2026-09-10"
}
```
- `title`, `amount` are required. `category` defaults to `Others` if omitted.
  `description` and `date` are optional (`date` defaults to today).

Response `201`: the created expense object.

### `GET /api/expenses`
List the authenticated user's expenses. **Auth required.**

Query parameters (all optional):

| Param       | Values                                   | Description                                  |
|-------------|-------------------------------------------|-----------------------------------------------|
| `filter`    | `week` \| `month` \| `3months` \| `custom` | Date-range filter                             |
| `startDate` | `YYYY-MM-DD`                              | Required when `filter=custom`                 |
| `endDate`   | `YYYY-MM-DD`                              | Required when `filter=custom`                 |
| `category`  | one of the allowed categories             | Filter by category                            |
| `page`      | integer, default `1`                      | Pagination page                                |
| `limit`     | integer, default `20`, max `100`          | Page size                                      |

Examples:
```
GET /api/expenses?filter=week
GET /api/expenses?filter=month&category=Groceries
GET /api/expenses?filter=custom&startDate=2026-01-01&endDate=2026-03-31
GET /api/expenses?page=2&limit=10
```

Response `200`:
```json
{
  "data": [ { "id": 1, "title": "Weekly groceries", "amount": 54.2, "...": "..." } ],
  "pagination": { "page": 1, "limit": 20, "total": 1, "totalPages": 1 }
}
```

### `GET /api/expenses/:id`
Get a single expense by id (must belong to the authenticated user). **Auth required.**

### `PUT /api/expenses/:id`
Update an existing expense. Send only the fields you want to change. **Auth required.**

### `DELETE /api/expenses/:id`
Delete an expense. Returns `204 No Content` on success. **Auth required.**

---

## Error responses

Errors are returned as JSON in the shape `{ "error": "message" }` or, for validation
failures, `{ "errors": [ ... ] }`. Common status codes: `400` (validation), `401`
(missing/invalid/expired token or bad credentials), `404` (not found), `409`
(duplicate email on signup), `500` (server error).

## Notes / possible extensions

- Swap SQLite for Postgres by changing `src/config/db.js` and adding the `pg`
  package — Sequelize models require no changes.
- Add refresh tokens / token revocation for longer-lived sessions.
- Add rate limiting (`express-rate-limit`) on `/api/auth/*` to slow brute-force attempts.
