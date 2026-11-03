# Real-Time Collaborative Kanban Board — Backend

Stack: Node.js, Express, PostgreSQL, JWT, bcrypt, Socket.IO.

## 1. Install

```bash
npm install
```

Copy `.env.example` to `.env`, then edit the PostgreSQL password and `JWT_SECRET`.

Windows CMD:
```cmd
copy .env.example .env
```

PowerShell:
```powershell
Copy-Item .env.example .env
```

Use a long random JWT secret. Do not commit `.env` to GitHub.

## 2. Database

This backend expects the PostgreSQL schema created earlier, including:
`organizations`, `tenants`, `users`, `memberships`, `boards`, `columns`, `tasks`, `activity_logs`.

The application uses UUID IDs and the column names in the schema described for this project. If you have not created those tables yet, use your project schema first.

## 3. Run

```bash
npm run dev
```

Test:
- `GET http://localhost:5000/api/health`
- `GET http://localhost:5000/api/health/db`

## 4. API overview

All routes below except register/login/health require:
`Authorization: Bearer YOUR_JWT_TOKEN`

- `POST /api/auth/register` — `{ "name", "email", "password", "organizationName", "tenantName" }`
- `POST /api/auth/login` — `{ "email", "password" }`
- `GET /api/auth/me`
- `GET /api/organizations`
- `GET /api/tenants` — list tenant memberships
- `POST /api/tenants` — `{ "organizationId", "name" }`, org admin only
- `GET /api/boards?tenantId=UUID`
- `POST /api/boards` — `{ "tenantId", "name", "description" }`
- `GET /api/boards/:boardId?tenantId=UUID`
- `PATCH /api/boards/:boardId` — `{ "tenantId", "name", "description" }`
- `DELETE /api/boards/:boardId?tenantId=UUID`
- `POST /api/boards/:boardId/columns` — `{ "tenantId", "name", "position" }`
- `PATCH /api/columns/:columnId` — `{ "tenantId", "name", "position" }`
- `DELETE /api/columns/:columnId?tenantId=UUID`
- `GET /api/tasks?tenantId=UUID&boardId=UUID`
- `POST /api/tasks` — `{ "tenantId", "boardId", "columnId", "title", "description", "priority", "position", "assigneeId", "dueAt" }`
- `PATCH /api/tasks/:taskId` — provide `tenantId` and fields to update; include `version` for optimistic concurrency
- `DELETE /api/tasks/:taskId?tenantId=UUID`
- `GET /api/activity?tenantId=UUID&limit=50`

## 5. Real-time Socket.IO

Connect to the same origin with Socket.IO and send the JWT:
```js
io("http://localhost:5000", {
  auth: { token: "YOUR_JWT_TOKEN" }
});
```

Join a board after the server checks membership:
```js
socket.emit("board:join", { tenantId: "...", boardId: "..." });
socket.on("board:joined", console.log);
socket.on("task:created", console.log);
socket.on("task:updated", console.log);
socket.on("task:deleted", console.log);
socket.on("column:created", console.log);
socket.on("column:updated", console.log);
socket.on("column:deleted", console.log);
```

Socket broadcasts happen after successful database commits. Clients must still fetch the REST API as the source of truth.

## Notes

- Registration creates a new organization and a first tenant for the new user.
- Email addresses are normalized to lowercase.
- This is a development-ready baseline, not a complete production deployment. Before production, add rate limiting, email verification, password reset, structured logging, request validation/rate limits, automated tests, migrations, and deployment secrets.
- PostgreSQL Row Level Security policies require transaction-local tenant context. This API relies on server-side authorization checks; if RLS is enabled and forced on your tables, you must set `app.tenant_id` within each tenant-scoped transaction. Do not expose database credentials or unrestricted SQL to clients.
