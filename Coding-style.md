# AGENTS.md — Project Build Rules

> Drop this at the repo root. Rename to `CLAUDE.md`, `.cursorrules`, or `.github/copilot-instructions.md` if your tool expects that name.
> **This file is normative.** If a generated file contradicts a rule here, the file is wrong.

---

## 0. How to read this document

| Keyword | Meaning |
|---|---|
| **MUST** | Non-negotiable. Violating it = broken deliverable. |
| **MUST NOT** | Never generate this, even if asked casually. |
| **DEFAULT** | Do this without being asked. |
| **OPT-IN** | Do this **only** if the user names it explicitly in the request. |

**Precedence:** explicit user instruction in the current request → this file → your own habits (last).

**Ambiguity rule:** if the user did not mention an OPT-IN item, do not add it, do not ask about it, and do not stub it. Build the default stack and list what was left out in the README's "Not included" section.

---

## 1. Rule 0 — Defaults vs Opt-in

| Concern | Status | Choice |
|---|---|---|
| Runtime | DEFAULT | Node.js (ESM, `"type": "module"`) |
| HTTP framework | DEFAULT | Express.js |
| ORM | DEFAULT | Drizzle ORM |
| Database | DEFAULT | PostgreSQL via `pg`, connected through a hosted connection string in `DATABASE_URL` |
| Validation | DEFAULT | Zod |
| Language | DEFAULT | **JavaScript** |
| Docs | DEFAULT | `README.md` + `design.md` (both required, both detailed) |
| Frontend | DEFAULT | Next.js (App Router) + shadcn/ui or Aceternity UI |
| TypeScript | **OPT-IN** | Only if the user says "TypeScript"/"TS" |
| Redis (`ioredis`) | **OPT-IN** | Only if the user says "Redis"/"cache". Hosted URL in `REDIS_URL` |
| BullMQ + worker | **OPT-IN** | Only if the user says "queue"/"BullMQ"/"background job" |
| Docker | **OPT-IN** | Only if the user says "Docker"/"containerize" |
| Auth module | **OPT-IN** | Only if the user says "auth"/"login"/"JWT" |

Never install a package that is not required by a DEFAULT or an enabled OPT-IN.

---

## 2. Repository layout

Exactly two top-level folders. No code at the repo root except config/docs.

```
<project>/
├── frontend/                 # Next.js app
├── backend/
│   ├── server.js             # THE ONLY ENTRY POINT
│   ├── package.json
│   ├── drizzle.config.js
│   ├── openapi.yaml          # API spec — mirrors the routers 1:1
│   ├── example.env           # committed placeholders (.env is gitignored)
│   ├── routers/              # <module>.js
│   ├── controllers/          # <module>.js
│   ├── services/             # <module>.js
│   ├── repositories/         # <module>.js
│   ├── middlewares/          # error.js, auth.js (auth only if OPT-IN)
│   ├── utils/                # asyncHandler.js, ApiError.js, ApiResponse.js, errorCodes.js, env.js
│   └── db/
│       ├── index.js          # exports `db` + `connectDb()`
│       ├── schema/           # <module>.js  (drizzle tables)
│       └── migrations/       # drizzle-kit output
├── README.md
└── design.md
```

**Module naming law:** one module = one name = the same filename in every layer.
`auth` → `routers/auth.js`, `controllers/auth.js`, `services/auth.js`, `repositories/auth.js`, `db/schema/auth.js`.
MUST NOT use `auth.controller.js`, `authController.js`, or plural/singular drift.

**Folder law:** the folders above are the complete allowed set for `backend/`. `middlewares/` exists because the global error handler and the auth middleware need a home. MUST NOT invent `helpers/`, `lib/`, `common/`, `types/`, `constants/`, `validators/` unless the user asks.

**Single entry point law:** there is exactly one `index.js` in the entire backend — `db/index.js`. MUST NOT create barrel `index.js` files in `routers/`, `services/`, etc. The app starts only from `server.js`.

---

## 3. The layer contract (the core of this document)

Data flows **down** one layer at a time and never skips a layer.

```
HTTP  →  router  →  controller  →  service  →  repository  →  Drizzle/Postgres
                       ↑ zod         ↑ business        ↑ pure SQL
                       ↓ ApiResponse ↓ ApiError        ↓ rows | null
```

| Layer | Receives | Returns | May import | MUST NOT |
|---|---|---|---|---|
| **router** | HTTP method + path | nothing | middlewares, its own controller | contain logic, validation, or `try/catch` |
| **controller** | `req, res` | JSON response | its service, zod, `ApiResponse`, `asyncHandler` | query the DB, hold business rules, call another controller |
| **service** | a plain validated object (DTO) | plain data | its repository, other services, `ApiError`, utils | touch `req`, `res`, `next`, cookies, headers, or write SQL |
| **repository** | primitives / DTO (+ optional `tx`) | rows or `null` | `db`, `schema`, drizzle operators | contain `if`-business-rules, throw `ApiError`, call a service |
| **middlewares** | `req, res, next` | `next()` or throws | utils | contain module logic |
| **utils** | — | — | nothing from routers/controllers/services/repositories | import any layer (keeps them dependency-free) |
| **db** | — | `db`, `connectDb`, schema | drizzle, pg | run queries |

**The req/res boundary law:** `req` and `res` stop at the controller. A service MUST receive a hand-built object containing only the fields it needs. This is what makes services testable and reusable by a worker or a CLI.

```js
// ❌ NEVER
await authService.register(req, res);
await authService.register(req.body);          // unvalidated, unknown shape

// ✅ ALWAYS — collated, validated, minimal
const { email, password } = registerSchema.parse(req.body);
const user = await authService.register({ email, password, ip: req.ip });
```

---

## 4. Request lifecycle — trace this exactly

`POST /api/v1/auth/register`

1. `server.js` → `express.json()` parses the body → `app.use("/api/v1/auth", authRouter)`.
2. `routers/auth.js` matches `POST /register` → `asyncHandler(authController.register)`.
3. `controllers/auth.js` parses `req.body` with a zod schema. Invalid → zod throws → caught by `asyncHandler` → `next(err)` → error middleware → `400` with field details.
4. Controller builds `{ email, password }` and calls `authService.register(payload)`.
5. `services/auth.js` applies rules: `authRepository.findByEmail(email)`; if found → `throw new ApiError(409)`. Hashes the password. Calls `authRepository.create({ email, passwordHash })`. Signs the JWT.
6. `repositories/auth.js` runs the Drizzle query and returns the row. It makes no decisions.
7. Service returns `{ user, token }` to the controller.
8. Controller sets the httpOnly cookie and responds `201` with `new ApiResponse(user, "Registered")`. **The token is never in the JSON body.**
9. Any throw anywhere lands in `middlewares/error.js`, the single place that converts errors into the standard error shape.

---

## 5. Reference implementations (copy these verbatim)

### 5.1 `utils/errorCodes.js`

```js
export const ERROR_CODES = {
  BAD_REQUEST: 400,
  UNAUTHORIZED: 401,
  FORBIDDEN: 403,
  NOT_FOUND: 404,
  CONFLICT: 409,
  UNPROCESSABLE: 422,
  TOO_MANY_REQUESTS: 429,
  INTERNAL: 500,
};

export const ERROR_MESSAGES = {
  400: "Bad Request",
  401: "Unauthorized",
  403: "Forbidden",
  404: "Not Found",
  409: "Conflict",
  422: "Unprocessable Entity",
  429: "Too Many Requests",
  500: "Internal Server Error",
};
```

### 5.2 `utils/ApiError.js`

```js
import { ERROR_MESSAGES } from "./errorCodes.js";

export class ApiError extends Error {
  constructor(statusCode = 500, message = ERROR_MESSAGES[statusCode], details = null) {
    super(message);
    this.statusCode = statusCode;
    this.details = details;        // e.g. zod field errors
    this.isOperational = true;     // false/absent ⇒ unexpected crash
    Error.captureStackTrace(this, this.constructor);
  }
}
```

Usage anywhere in a service: `throw new ApiError(404, "User not found");` or just `throw new ApiError(404);`

### 5.3 `utils/ApiResponse.js`

```js
export class ApiResponse {
  constructor(data = null, message = "Success") {
    this.success = true;
    this.error = false;
    this.message = message;
    this.data = data;
  }
}
```

### 5.4 `utils/asyncHandler.js`

```js
export const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);
```

Every controller and every async middleware is wrapped in this. **MUST NOT write `try/catch` inside a controller.** A `try/catch` is allowed in a service only to translate a low-level error into an `ApiError`, and it must rethrow.

### 5.5 `middlewares/error.js` — registered last in `server.js`

```js
import { ZodError } from "zod";
import { ApiError } from "../utils/ApiError.js";
import { ERROR_MESSAGES } from "../utils/errorCodes.js";

export const notFound = (req, _res, next) =>
  next(new ApiError(404, `Route ${req.method} ${req.originalUrl} not found`));

// eslint-disable-next-line no-unused-vars
export const errorHandler = (err, _req, res, _next) => {
  let statusCode = 500;
  let message = ERROR_MESSAGES[500];
  let details = null;

  if (err instanceof ZodError) {
    statusCode = 400;
    message = "Validation failed";
    details = err.issues.map((i) => ({ field: i.path.join("."), message: i.message }));
  } else if (err instanceof ApiError) {
    statusCode = err.statusCode;
    message = err.message;
    details = err.details;
  }

  if (statusCode >= 500) console.error(err);

  res.status(statusCode).json({
    success: false,
    error: true,
    message,
    data: null,
    ...(details && { details }),
  });
};
```

### 5.6 `db/index.js`

```js
import { drizzle } from "drizzle-orm/node-postgres";
import pkg from "pg";
import * as schema from "./schema/index.js"; // the one permitted extra: schema re-export
import { env } from "../utils/env.js";

const { Pool } = pkg;

export const pool = new Pool({
  connectionString: env.DATABASE_URL,
  ssl: { rejectUnauthorized: false }, // hosted Postgres (Neon/Supabase/Railway)
});

export const db = drizzle(pool, { schema });

export const connectDb = async () => {
  await pool.query("SELECT 1"); // fail fast if the hosted DB is unreachable
};
```

### 5.7 `server.js` — the only entry point

```js
import express from "express";
import cors from "cors";
import cookieParser from "cookie-parser";

import { env } from "./utils/env.js";
import { connectDb } from "./db/index.js";
import { notFound, errorHandler } from "./middlewares/error.js";
import authRouter from "./routers/auth.js";
import userRouter from "./routers/user.js";

const app = express();

app.use(cors({ origin: env.CLIENT_URL, credentials: true }));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

app.get("/health", (_req, res) => res.json({ success: true, error: false, message: "OK", data: null }));

/* ---- route declarations: every router of the app is mounted here ---- */
app.use("/api/v1/auth", authRouter);
app.use("/api/v1/users", userRouter);
/* -------------------------------------------------------------------- */

app.use(notFound);
app.use(errorHandler); // MUST be last

const startServer = async () => {
  try {
    await connectDb();
    console.log("✅ DB started");
    app.listen(env.PORT, () => console.log(`🚀 Server listening on http://localhost:${env.PORT}`));
  } catch (err) {
    console.error("❌ Failed to start server:", err);
    process.exit(1);
  }
};

startServer();
```

`server.js` MUST NOT contain business logic, inline route handlers (except `/health` and the optional `/docs` mount from §12), or DB queries.

### 5.8 `utils/env.js` — validate the environment with zod, once

```js
import { z } from "zod";
import "dotenv/config";

const envSchema = z.object({
  PORT: z.coerce.number().default(8000),
  DATABASE_URL: z.string().url(),
  CLIENT_URL: z.string().url().default("http://localhost:3000"),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  // JWT_SECRET / REDIS_URL are added only when those modules are opted in
});

export const env = envSchema.parse(process.env);
```

---

## 6. Vertical slice — the shape every module copies

### `routers/user.js`
```js
import { Router } from "express";
import { asyncHandler } from "../utils/asyncHandler.js";
import { requireAuth } from "../middlewares/auth.js";
import * as userController from "../controllers/user.js";

const router = Router();

router.get("/me", requireAuth, asyncHandler(userController.getMe));
router.patch("/me", requireAuth, asyncHandler(userController.updateMe));

export default router;
```
Routes only. No `app`, no logic, no `/api/v1` prefix (that lives in `server.js`).

### `controllers/user.js`
```js
import { z } from "zod";
import { ApiResponse } from "../utils/ApiResponse.js";
import * as userService from "../services/user.js";

const updateMeSchema = z.object({
  name: z.string().min(2).max(60).optional(),
  bio: z.string().max(280).optional(),
});

export const getMe = async (req, res) => {
  const user = await userService.getProfile({ userId: req.user.id });
  res.status(200).json(new ApiResponse(user, "Profile fetched"));
};

export const updateMe = async (req, res) => {
  const payload = updateMeSchema.parse(req.body);           // body
  const user = await userService.updateProfile({ userId: req.user.id, ...payload });
  res.status(200).json(new ApiResponse(user, "Profile updated"));
};
```
Validate **every** untrusted input — `body`, `params`, `query` — each with its own schema. Never `parse(req)`.

### `services/user.js`
```js
import { ApiError } from "../utils/ApiError.js";
import * as userRepository from "../repositories/user.js";

export const getProfile = async ({ userId }) => {
  const user = await userRepository.findById(userId);
  if (!user) throw new ApiError(404, "User not found");
  return sanitize(user);
};

export const updateProfile = async ({ userId, ...fields }) => {
  if (Object.keys(fields).length === 0) throw new ApiError(400, "No fields to update");
  const updated = await userRepository.updateById(userId, fields);
  if (!updated) throw new ApiError(404, "User not found");
  return sanitize(updated);
};

const sanitize = ({ passwordHash, ...safe }) => safe; // never leak secrets upward
```

### `repositories/user.js`
```js
import { eq } from "drizzle-orm";
import { db } from "../db/index.js";
import { users } from "../db/schema/user.js";

export const findById = async (id, tx = db) => {
  const [row] = await tx.select().from(users).where(eq(users.id, id)).limit(1);
  return row ?? null;
};

export const updateById = async (id, fields, tx = db) => {
  const [row] = await tx.update(users)
    .set({ ...fields, updatedAt: new Date() })
    .where(eq(users.id, id))
    .returning();
  return row ?? null;
};
```

**Transaction law:** transactions are opened in the **service** and the `tx` handle is passed into repository functions, which all accept `tx = db` as their last parameter.

```js
await db.transaction(async (tx) => {
  const order = await orderRepository.create(payload, tx);
  await inventoryRepository.decrement(payload.items, tx);
  return order;
});
```

---

## 7. Response & error contract (the API's public promise)

Success — always this shape, at every endpoint:
```json
{ "success": true, "error": false, "message": "Profile fetched", "data": { } }
```

Failure — always this shape, produced only by `middlewares/error.js`:
```json
{ "success": false, "error": true, "message": "Bad Request", "data": null,
  "details": [{ "field": "email", "message": "Invalid email" }] }
```

Status code discipline: `200` read/update · `201` create · `204` delete-with-no-body (still send the standard body instead if the client needs a message) · `400` validation · `401` missing/bad credentials · `403` authenticated but not allowed · `404` missing resource · `409` duplicate/state conflict · `500` unexpected.

MUST NOT: `res.send("ok")`, bare `res.json(user)`, or a different envelope in any single route.

---

## 8. Auth module (OPT-IN — build only when asked)

- Classic JWT. `jsonwebtoken` + `bcrypt` (cost 10–12).
- **The token is delivered only as an httpOnly cookie.** MUST NOT return it in the response body or the `data` field.
  ```js
  res.cookie("accessToken", token, {
    httpOnly: true,
    secure: env.NODE_ENV === "production",
    sameSite: env.NODE_ENV === "production" ? "none" : "lax",
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });
  ```
- CORS MUST use `credentials: true` and an explicit origin (never `*`).
- Logout clears the cookie with the same options.
- `middlewares/auth.js` exports `requireAuth`: read `req.cookies.accessToken` → `jwt.verify` → attach `req.user = { id, role }` → `next()`. Missing/expired/invalid → `throw new ApiError(401)`. Optional `requireRole(...roles)` → `403`.
- Passwords are hashed in the **service**, never in the controller or repository. Password hashes never cross the service boundary upward.

### 8.1 Dev-mode header auth (so the API is testable in Postman/curl)

Cookies are painful in API clients, so **in non-production only** the same JWT may be sent as `Authorization: Bearer <token>`. This needs both halves — a way to *get* the token, and a way to *send* it.

**Half 1 — issue it.** On login/register, always set the cookie; additionally include the raw token in `data` when not in production:

```js
res.cookie("accessToken", token, cookieOptions);
res.status(200).json(new ApiResponse(
  {
    user,
    // DEV ONLY — copy into `Authorization: Bearer <token>`. Absent in production.
    ...(env.NODE_ENV !== "production" && { token }),
  },
  "Logged in",
));
```

**Half 2 — accept it.** `middlewares/auth.js` prefers the cookie and falls back to the header only outside production:

```js
import jwt from "jsonwebtoken";
import { ApiError } from "../utils/ApiError.js";
import { env } from "../utils/env.js";

const allowHeaderAuth = env.NODE_ENV !== "production";

const extractToken = (req) => {
  if (req.cookies?.accessToken) return req.cookies.accessToken;      // browsers, always
  if (allowHeaderAuth) {                                            // API clients, dev only
    const header = req.headers.authorization;
    if (header?.startsWith("Bearer ")) return header.slice(7);
  }
  return null;
};

export const requireAuth = (req, _res, next) => {
  const token = extractToken(req);
  if (!token) throw new ApiError(401, "Authentication required");

  try {
    const { sub, role } = jwt.verify(token, env.JWT_SECRET);
    req.user = { id: sub, role };
    return next();
  } catch {
    throw new ApiError(401, "Invalid or expired token");
  }
};
```

Rules for the fallback:
- Gated on `env.NODE_ENV !== "production"` and nothing else — one variable is the kill switch, so there is no way to leave it half-on.
- The cookie always wins when both are present; the header is never the primary path.
- Same secret, same signing, same `requireAuth` — dev and prod differ only in *where the token is read from*, so a test in Postman proves the real code path.
- `server.js` logs a one-line warning at boot when it is active: `⚠️  Header (Bearer) auth enabled — development only`.
- This middleware is synchronous, so Express forwards its throws automatically; do not wrap it in `asyncHandler`.
- Document it in the README under a "Testing protected routes" heading, and mark `bearerAuth` as dev-only in `openapi.yaml`.

---

## 9. Opt-in modules

**Redis (`ioredis`)** — `utils/redis.js` exports one shared client built from `REDIS_URL`; log on `connect` and `error`. Only services use it. Cache keys are namespaced `entity:id`. Every cached write invalidates its key.

**BullMQ** — `queues/<name>.js` (queue definition) and `workers/<name>.js` (processor). The worker is a **separate process** with its own npm script (`"worker": "node workers/<name>.js"`) — never started inside `server.js`. Processors call the same services the HTTP layer uses; that is the payoff of the req/res boundary law. Document retries, backoff, and idempotency in `design.md`.

**TypeScript** — every rule above holds with `.ts`. Add `types` inline in the layer that owns them; DTOs are inferred from zod (`z.infer<typeof schema>`). Repositories use Drizzle's `InferSelectModel`/`InferInsertModel`. No `any`.

**Docker** — `backend/Dockerfile` (multi-stage, non-root, `NODE_ENV=production`), `frontend/Dockerfile`, root `docker-compose.yml` wiring frontend + backend (+ redis if enabled). Hosted Postgres stays external; do not add a `postgres` service unless asked. All secrets via env, none baked into the image.

---

## 10. Frontend rules (Next.js)

- App Router. Plain, unclever Next.js — no custom state-management library unless asked.
- UI from **shadcn/ui** or **Aceternity UI**. Generated primitives live in `components/ui/`; your own composites in `components/`.
- Tailwind for styling. No parallel CSS-in-JS system.
- `lib/api.js` is the single fetch wrapper: base URL from `NEXT_PUBLIC_API_URL`, `credentials: "include"` on every call, unwraps the `{ success, error, message, data }` envelope, throws on `error: true`. Components MUST NOT call `fetch` directly.
- Server Components by default; `"use client"` only for interactivity.
- `app/` = routing and data fetching. `components/` = presentation. No business logic in JSX.
- Never read a cookie-based token in JS — it is httpOnly by design.

---

## 11. Environment — `example.env` (always produced)

`backend/example.env` is committed; `backend/.env` is gitignored and never generated with real values.

**Sync law:** `example.env` ⇄ `utils/env.js` ⇄ the README env table are bidirectional. Every key appears in all three, or in none. Adding a variable to the zod schema without adding it to `example.env` is a defect.

**Placeholder law:** values are shape-correct examples, never working secrets. `JWT_SECRET=replace-with-a-long-random-string`, not an actual key.

**Sectioning law:** group by module and mark which OPT-IN turns each block on. Blocks for modules that were not built are omitted entirely — not commented out.

```dotenv
# ---------- Core (always required) ----------
PORT=8000
NODE_ENV=development                     # development | production | test
DATABASE_URL=postgresql://user:password@ep-xxx.aws.neon.tech/dbname?sslmode=require
CLIENT_URL=http://localhost:3000         # exact origin, used by CORS with credentials

# ---------- Auth (only when the auth module is enabled) ----------
JWT_SECRET=replace-with-a-long-random-string
JWT_EXPIRES_IN=7d
COOKIE_NAME=accessToken

# ---------- Redis (only when Redis is enabled) ----------
REDIS_URL=rediss://default:password@host.upstash.io:6379

# ---------- BullMQ (only when the queue is enabled) ----------
QUEUE_NAME=default
WORKER_CONCURRENCY=5
```

`frontend/example.env`:

```dotenv
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
```

The app MUST fail loudly at boot if a required variable is missing — that is what `envSchema.parse(process.env)` in §5.8 buys you. Never `process.env.FOO || "fallback"` at a call site.

---

## 12. API spec — `openapi.yaml` (always produced)

`backend/openapi.yaml`, OpenAPI **3.1.0**. This is the contract the frontend, Postman, and any client codegen read.

**Sync law (bidirectional):** every route mounted in `server.js` + `routers/*` has exactly one path/operation entry, and every entry maps to a real route. A route added without a spec entry is an incomplete feature. `operationId` MUST equal `<module>.<controllerExport>` (e.g. `user.updateMe`) so the spec and the code are greppable from each other.

**Schema law:** request/response schemas mirror the zod schemas field-for-field, including `minLength`, `format`, and enums. If a zod schema changes, the yaml changes in the same edit.

**Envelope law:** success and error bodies are declared once in `components` and referenced everywhere via `$ref`/`allOf`. No endpoint redefines the envelope inline.

**Coverage law:** each operation declares `tags` (module name), `summary`, `security` (`security: []` for public routes), `requestBody` where applicable, and *every realistic* response — the happy path plus each `ApiError` the service can throw (400/401/403/404/409/429/500).

Required skeleton:

```yaml
openapi: 3.1.0
info:
  title: <Project> API
  version: 1.0.0
  description: Data flow, layering and edge cases are documented in design.md.
servers:
  - url: http://localhost:8000/api/v1
    description: Local
  - url: https://api.example.com/api/v1
    description: Production

tags:
  - name: auth
    description: Registration, login, session
  - name: users
    description: Profile read/write

security:                       # applied to every operation unless overridden
  - cookieAuth: []
  - bearerAuth: []              # development/testing only

paths:
  /auth/login:
    post:
      tags: [auth]
      operationId: auth.login
      summary: Log in and receive an httpOnly accessToken cookie
      security: []              # public
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/LoginRequest' }
      responses:
        '200':
          description: >
            Logged in. Sets the accessToken cookie. Outside production the body
            also carries `token` for Authorization: Bearer testing.
          headers:
            Set-Cookie:
              schema:
                type: string
                example: accessToken=<jwt>; HttpOnly; Path=/; SameSite=Lax
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ApiResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          user: { $ref: '#/components/schemas/User' }
                          token:
                            type: string
                            description: Development only. Absent in production.
        '400': { $ref: '#/components/responses/BadRequest' }
        '401': { $ref: '#/components/responses/Unauthorized' }

  /users/me:
    get:
      tags: [users]
      operationId: user.getMe
      summary: Fetch the authenticated user's profile
      responses:
        '200':
          description: Profile fetched
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ApiResponse'
                  - type: object
                    properties:
                      data: { $ref: '#/components/schemas/User' }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '404': { $ref: '#/components/responses/NotFound' }

components:
  securitySchemes:
    cookieAuth:
      type: apiKey
      in: cookie
      name: accessToken
      description: Primary mechanism. Set by the server, httpOnly.
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: Accepted only when NODE_ENV !== production. For API clients.

  schemas:
    ApiResponse:
      type: object
      required: [success, error, message, data]
      properties:
        success: { type: boolean, const: true }
        error: { type: boolean, const: false }
        message: { type: string }
        data: {}
    ErrorResponse:
      type: object
      required: [success, error, message, data]
      properties:
        success: { type: boolean, const: false }
        error: { type: boolean, const: true }
        message: { type: string, example: Bad Request }
        data: { type: 'null' }
        details:
          type: array
          description: Present on validation failures.
          items:
            type: object
            properties:
              field: { type: string, example: email }
              message: { type: string, example: Invalid email }
    LoginRequest:
      type: object
      required: [email, password]
      properties:
        email: { type: string, format: email }
        password: { type: string, minLength: 8 }
    User:
      type: object
      properties:
        id: { type: string, format: uuid }
        email: { type: string, format: email }
        name: { type: string }
        createdAt: { type: string, format: date-time }

  responses:
    BadRequest:
      description: Validation failed
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
    Unauthorized:
      description: Missing or invalid credentials
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
    Forbidden:
      description: Authenticated but not permitted
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
    Conflict:
      description: Resource already exists or state conflict
      content:
        application/json:
          schema: { $ref: '#/components/schemas/ErrorResponse' }
```

**Optional serving:** `swagger-ui-express` mounted at `/api/v1/docs` in `server.js`, gated to non-production, reading the yaml with `yaml`/`js-yaml`. This is the only route besides `/health` allowed to be declared inline there.

---

## 13. Required documentation (always produced, never a stub)

### `README.md`
1. What the project does, in 3 lines.
2. Tech stack table (and a **"Not included"** line naming the OPT-INs that were skipped).
3. Folder structure tree with a one-line purpose per folder.
4. Prerequisites, install, and run steps for `backend/` and `frontend/` separately.
5. Full env table mirroring `example.env`: variable · required? · example · what it's for · which module needs it.
6. Drizzle workflow: `generate`, `migrate`, `studio` commands.
7. **Complete route reference**: method · path · auth required · request body/params · success response · error codes — and a pointer to `openapi.yaml` as the machine-readable source of truth.
8. Sample success and error response bodies.
9. **Testing protected routes**: log in, copy `data.token` from the dev response, send it as `Authorization: Bearer <token>`; state plainly that this works only when `NODE_ENV !== production`. Include a curl example.
10. Scripts table, troubleshooting, and the worker/Docker commands if enabled.

### `design.md`
1. **System design / HLD**: component diagram (ASCII or mermaid) — client, API, Postgres, and Redis/worker if enabled — and how each component communicates (protocol, sync/async).
2. Why this layering exists and what each layer is allowed to know (restate the contract).
3. **Per-route data flow**: for every endpoint, walk router → controller (which schema) → service (which rules) → repository (which query) → response.
4. Data model: tables, columns, types, indexes, relations, and the reasoning behind them.
5. **Edge cases, exhaustively**: duplicate/idempotent requests, concurrent writes and where the transaction boundary sits, partial failures, missing/expired token, expired cookie during a request, pagination limits, empty result sets, oversized payloads, invalid enum values, timezone/`Date` handling, DB connection loss at boot vs at runtime, cascade behaviour on delete, rate-limit behaviour, and — if auth is enabled — why the Bearer fallback cannot be reached in production and what happens if both a cookie and a header are sent.
6. Error taxonomy: which code is thrown where, and what the client should do about it.
7. Scaling notes: what is stateless, what would be cached, what would move to a queue, and known trade-offs.
8. If Redis/BullMQ are enabled: cache keys, TTLs, invalidation strategy, job payload shape, retry/backoff policy, idempotency guarantees.

---

## 14. Conventions

- ESM everywhere (`"type": "module"`), `.js` extensions in relative imports.
- Named exports from controllers/services/repositories; **default export only for routers**.
- `camelCase` variables and functions, `PascalCase` classes, `SCREAMING_SNAKE_CASE` constants, `snake_case` Postgres columns mapped to `camelCase` in the Drizzle schema.
- Route paths are kebab-case plural nouns: `/api/v1/order-items`.
- Every table gets `id`, `createdAt`, `updatedAt`. Prefer `uuid` primary keys.
- Async/await only. No `.then()` chains, no callbacks.
- Comments explain *why*, never *what*.
- Version the API: everything mounts under `/api/v1`.

---

## 15. Anti-patterns — reject on sight

1. Passing `req`/`res` into a service.
2. A DB query outside `repositories/`.
3. `try/catch` inside a controller.
4. Business rules (`if (user.balance < amount)`) inside a repository.
5. A second `index.js` or a barrel file anywhere.
6. Two different response shapes in one API.
7. Building an error response outside `middlewares/error.js`.
8. A JWT in the response body.
9. `cors({ origin: "*" })` with credentials.
10. Adding Redis, BullMQ, TypeScript, Docker, or auth without an explicit request.
11. Unvalidated `req.body` reaching a service.
12. `process.env.X` read outside `utils/env.js`.
13. A generated `README.md` or `design.md` that is a placeholder.
14. Route handlers written inline in `server.js`.
15. Validation logic duplicated in the service after the controller already ran zod.
16. A route that exists in `routers/` but not in `openapi.yaml` (or the reverse), or a spec envelope redefined inline instead of `$ref`-ed.
17. Returning the JWT in the body without the `NODE_ENV !== "production"` guard, or gating the Bearer fallback on anything other than that one check.
18. A real secret in `example.env`, a `.env` that isn't gitignored, or an env key read outside `utils/env.js`.

---

## 16. Definition of done

- [ ] `frontend/` and `backend/` exist; nothing else at root but docs/config.
- [ ] Every module has the same filename across router/controller/service/repository.
- [ ] `server.js` is the sole entry point and declares every router.
- [ ] `startServer()` connects the DB, logs DB started, then listens.
- [ ] `asyncHandler`, `ApiError`, `ApiResponse`, `errorCodes`, error middleware all present and used.
- [ ] Every route returns the standard envelope; every failure passes through the error middleware.
- [ ] Every input is zod-validated in the controller and forwarded as a collated object.
- [ ] `example.env` matches `utils/env.js` and the README env table exactly, with placeholder values only.
- [ ] `openapi.yaml` covers every route, both directions, with shared envelope components and `operationId` matching the controller exports.
- [ ] If auth is enabled: cookie is the primary path, the Bearer fallback and the token-in-body are both gated on `NODE_ENV !== "production"`, and the boot warning is logged.
- [ ] Drizzle schema + migrations generated; `drizzle.config.js` points at `db/schema` and `db/migrations`.
- [ ] `README.md` and `design.md` complete, including the full route table, the "Testing protected routes" section, and the edge-case section.
- [ ] Zero OPT-IN dependencies present unless the user asked for them.
