# Backend — StockFlow API

REST API deployed to Cloudflare Workers using Hono.js, Drizzle ORM, Neon PostgreSQL, and Zod.

## Stack

| Layer         | Technology                                                          |
| ------------- | ------------------------------------------------------------------- |
| Runtime       | Cloudflare Workers (edge)                                           |
| Framework     | Hono.js                                                             |
| Database      | Neon PostgreSQL (serverless HTTP driver)                            |
| ORM           | Drizzle ORM                                                         |
| Validation    | Zod v4                                                              |
| Cache         | In-memory API cache at the service layer                            |
| Password hash | Web Crypto API — PBKDF2, SHA-256, 100k iterations, no external deps |
| Docs          | `@hono/zod-openapi` + `@hono/swagger-ui`                            |
| Local dev     | Wrangler + Miniflare                                                |
| Testing       | Vitest + Miniflare E2E                                              |

## Project Structure

```
src/
  index.ts          # Hono app entry point — mounts all routers
  controllers/      # HTTP only: parse request, call service, return response. No business logic.
  services/         # Business logic + API cache layer. Framework-agnostic.
  repositories/     # Data access only: Drizzle queries, mapping to domain objects. No business logic.
  models/           # Zod schemas, TypeScript interfaces, domain error definitions
  middleware/       # Hono middleware (auth, correlation ID, error handler)
  config/           # Env var validation (Zod) and typed constants — ONLY place c.env is read
  db/
    schema.ts       # Drizzle table definitions
    migrations/     # SQL migration files (UTC timestamp prefix: YYYYMMDDHHMMSS_description.sql)
    client.ts       # Drizzle + Neon HTTP client setup
  lib/
    cache.ts        # Service-layer in-memory cache utility
```

## Commands

```bash
pnpm dev          # wrangler dev — starts local Cloudflare Workers server
pnpm test         # vitest run
pnpm test:e2e     # E2E tests against Miniflare local state
pnpm lint         # eslint
pnpm format       # prettier --write
pnpm db:generate  # drizzle-kit generate — create migration from schema changes
pnpm db:migrate   # drizzle-kit migrate — apply pending migrations
pnpm deploy       # wrangler deploy — deploy to Cloudflare Workers
```

After `pnpm dev`, the API docs are available at:

| URL                                   | Description           |
| ------------------------------------- | --------------------- |
| `http://localhost:8787/api/docs`      | Swagger UI            |
| `http://localhost:8787/api/docs/spec` | OpenAPI 3.1 JSON spec |

## REST API Conventions

### URL Format

- Pattern: `/api/v{n}/resource-name`
- Resources are **plural kebab-case nouns**: `/api/v1/stock-items`, `/api/v1/purchase-orders`
- **Never use verbs** in paths: `/api/v1/orders/{id}/cancel` not `/api/v1/cancelOrder`
- Nested resources only one level deep: `/api/v1/orders/{id}/items`
- Versioning always in the URL path — never in headers or query params

### HTTP Status Codes

| Situation                          | Code |
| ---------------------------------- | ---- |
| Successful GET or PUT              | 200  |
| Resource created (POST)            | 201  |
| Successful DELETE (no body)        | 204  |
| Bad input (missing/invalid fields) | 400  |
| Not authenticated                  | 401  |
| Authenticated but not authorized   | 403  |
| Resource not found                 | 404  |
| State conflict (duplicate, locked) | 409  |
| Semantic validation failure        | 422  |
| Downstream dependency failed       | 502  |
| Unhandled server error             | 500  |

## Architecture Rules

### Layering (strict — never skip layers)

```
Controller → Service → Repository → Drizzle → Neon PostgreSQL
```

- **Controllers**: Parse request, validate with Zod schemas from `models/`, call exactly one service method, return response. No business logic. Never import from `repositories/`.
- **Services**: All business logic and orchestration. Framework-agnostic — never import from Hono. Never construct HTTP responses. May call multiple repositories.
- **Repositories**: Execute Drizzle queries, map results to domain types. No business logic. Return domain objects, never raw DB rows.

### Config Module (env vars)

All environment variables are read and validated **only** in `src/config/index.ts`. Never access `c.env` or `process.env` in controllers, services, or repositories.

```typescript
// src/config/index.ts — the ONLY place c.env is read
import { z } from "zod";

const schema = z.object({
  DATABASE_URL: z.url(),
  NODE_ENV: z.enum(["development", "test", "production"]),
});

export const config = schema.parse(env);
```

- Fail fast at startup if required env vars are missing — never silently fall back
- Boolean flags: `FEATURE_X_ENABLED=true` — parse as strings, convert explicitly
- Variable naming: `SCREAMING_SNAKE_CASE`, prefixed by owning service for cross-service vars
- `getConfig(env)` accepts `Record<string, unknown>` — **do not change this to `Record<string, string>`**. The `Bindings` type includes `BUCKET: R2Bucket` (a non-string service binding), so a narrower type causes a TypeScript error. Zod's `safeParse` handles `unknown` correctly.

### Zod v4 Schemas (models/)

Zod v4 breaking changes — always use the new API:

```typescript
// ❌ Zod 3 (OLD — do not use)
z.string().email();
z.string().uuid();
z.string().url();
z.string().nonempty();
z.object({ name: z.string() }).required_error("Required");

// ✅ Zod 4 (NEW — always use this)
z.email();
z.uuid();
z.url();
z.string().min(1);
z.object({ name: z.string() }, { error: "Required" });
```

- Use `z.infer<typeof Schema>` for TypeScript types — never define types separately
- Validate at the controller level — never trust raw request body in services or repositories
- Error messages use `{ error: "..." }` not `{ message: "..." }`

### Password Hashing

Passwords are hashed using the **Web Crypto API** (PBKDF2) — no Node.js `crypto`, no external libraries.
This is required because Cloudflare Workers does not support Node.js built-ins.

- Implementation: `src/lib/password.ts`
- Algorithm: PBKDF2, SHA-256, 100,000 iterations, 256-bit output
- Stored format: `"{saltBase64}:{hashBase64}"`
- Always use constant-time comparison (`verifyPassword`) — never compare hashes with `===`
- Never log, return, or expose the stored hash

### OpenAPI / Swagger Documentation

Every endpoint and every schema **must** be documented with OpenAPI. Use the official Hono packages:

```bash
pnpm add @hono/zod-openapi @hono/swagger-ui
```

#### Router setup

Replace `new Hono<AppContext>()` with `new OpenAPIHono<AppContext>()` in every controller:

```typescript
import { OpenAPIHono } from "@hono/zod-openapi";

export const usersRouter = new OpenAPIHono<AppContext>();
```

#### Route definition

Use `createRoute` to define each endpoint with its request/response schemas:

```typescript
import { createRoute, z } from "@hono/zod-openapi";

const getUser = createRoute({
  method: "get",
  path: "/{id}",
  tags: ["Users"],
  summary: "Get a user by ID",
  request: {
    params: z.object({ id: z.uuid() }),
  },
  responses: {
    200: {
      content: { "application/json": { schema: UserResponseSchema } },
      description: "User found",
    },
    404: {
      content: { "application/json": { schema: ErrorResponseSchema } },
      description: "User not found",
    },
  },
});

usersRouter.openapi(getUser, async (c) => { ... });
```

#### Swagger UI endpoint

Register the spec and the UI in `src/index.ts`:

```typescript
import { swaggerUI } from "@hono/swagger-ui";

app.doc("/api/docs/spec", {
  openapi: "3.1.0",
  info: { title: "StockFlow API", version: "1.0.0" },
});

app.get("/api/docs", swaggerUI({ url: "/api/docs/spec" }));
```

#### Rules

- Every `createRoute` must declare **all possible response codes** (200/201/204/400/404/409/422/500 as applicable)
- Request body, path params, and query params are all validated via Zod schemas in `createRoute` — do NOT duplicate `safeParse` calls
- Schemas in `models/` are reused directly — never define inline schemas in route definitions
- Tags group routes by resource: `["Users"]`, `["Products"]`, `["Sales"]`, etc.
- Summary is a short imperative phrase: `"List all users"`, `"Create a product"`

### API Cache (service layer)

- Cache sits inside service methods — NOT in controllers or repositories
- Cache key pattern: `resource:id` or `resource:list:hash(filters)`
- Always invalidate on mutations (create, update, delete)
- Always use TTL — never cache indefinitely

### CORS

CORS middleware is registered in `src/index.ts` **before all other middleware**, including auth. This ensures preflight `OPTIONS` requests are handled without hitting the auth check.

```typescript
import { cors } from "hono/cors";

const ALLOWED_ORIGINS = ["http://localhost:5173"]; // add deployed frontend URL here

app.use("*", cors({
  origin: (origin) => (ALLOWED_ORIGINS.includes(origin) ? origin : null),
  allowMethods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
  allowHeaders: ["Content-Type", "X-Correlation-Id"],
  exposeHeaders: ["X-Correlation-Id"],
  credentials: true, // required — auth uses httpOnly cookies
  maxAge: 600,
}));
```

Rules:
- `credentials: true` is mandatory — without it the browser will not send cookies cross-origin
- Never use `origin: "*"` with `credentials: true` — browsers block it; always specify exact origins
- Add the deployed frontend URL to `ALLOWED_ORIGINS` before each production deploy
- The origin function returns `null` (not `"*"`) for disallowed origins — Hono treats `null` as a CORS rejection

### Cloudflare Workers Constraints

- No Node.js APIs — use Web Standard APIs only (fetch, crypto, Request, Response)
- No long-running tasks — Workers have CPU time limits
- Secrets via `wrangler.toml` and `c.env` — accessed only through `config/index.ts`
- Neon uses HTTP driver (`@neondatabase/serverless`) — not standard TCP pg connection

### R2 Object Storage

**Bucket**: `stockflow` — binding name `BUCKET` — public URL `https://pub-f0bcf28b115849ffbbb6ac15fb70a6c2.r2.dev`

#### Configuration

- R2 binding declared in `wrangler.jsonc` under `r2_buckets`
- Public base URL stored as a non-secret var `R2_PUBLIC_URL` in `wrangler.jsonc` and validated in `src/config/index.ts`
- `BUCKET` is an `R2Bucket` service binding — it is **not** a string and cannot go through `getConfig()`. Access it directly from `c.env.BUCKET` in the controller only, with a comment explaining why

#### Upload endpoint pattern

- Route: `POST /api/v1/<resource>/images` — must be declared **before** `/:id` routes to avoid conflict
- Request: `multipart/form-data` with a `file` field — parse with `c.req.parseBody()`
- Validate in the service (not the controller): allowed MIME types, max file size
- Key format: `<resource>/{crypto.randomUUID()}.{ext}` where `ext` is derived from the MIME type
- Upload with `bucket.put(key, await file.arrayBuffer(), { httpMetadata: { contentType: file.type } })`
- Return `{ key, url }` where `url = \`${r2BaseUrl}/${key}\``
- Pass `r2BaseUrl` as a parameter to the service method — never access `c.env` inside a service

```typescript
// ✅ Correct layering — controller reads c.env, passes to service
const result = await service.uploadImage(c.env.BUCKET, file, config.R2_PUBLIC_URL);

// ❌ Wrong — service must never import or access c.env
```

#### Serving images

Images are served directly from R2's public URL — no Worker proxy needed:
```
https://pub-f0bcf28b115849ffbbb6ac15fb70a6c2.r2.dev/<key>
```

#### Upload-then-save flow (Option A)

The frontend uploads the file first, receives `{ key }`, then includes `imageKey` in the
create/update request. If the entity save fails, the uploaded object becomes orphaned in R2
(acceptable — clean up with an R2 lifecycle rule if needed). Never try to roll back R2 uploads
on application errors.

### Database Patterns

- All SQL uses Drizzle ORM — never raw string interpolation
- N+1 prevention: when fetching a list with related data, always batch — never fetch per item
- Never return raw DB errors to the client — map all DB errors to domain errors in repositories

#### Multi-table writes — NEVER use `db.transaction()`

The `neon-http` driver **does not support transactions** — calling `db.transaction()` throws
`Error: No transactions support in neon-http driver` at runtime, which surfaces as `INTERNAL_SERVER_ERROR`.

Use `db.batch()` instead. Neon executes a batch in a single HTTP round-trip inside an implicit
transaction, giving the same atomicity guarantee:

```typescript
// ✅ Correct — atomic multi-table write with db.batch()
await db.batch([
  db.insert(receipts).values({ ... }).returning(),
  ...items.map((item) =>
    db.update(products)
      .set({ stock: sql`${products.stock} + ${item.quantity}` })
      .where(eq(products.id, item.productId)),
  ),
] as const);

// ❌ Wrong — throws at runtime with neon-http
await db.transaction(async (tx) => {
  await tx.insert(receipts).values({ ... });
  await tx.update(products).set({ ... });
});
```

**Pattern for operations where writes depend on a prior insert's returned ID:**

```typescript
// 1. Insert the parent record to get its ID
const [parentRow] = await db.insert(parent).values({ ... }).returning();

// 2. Batch the child inserts + any side-effect updates
await db.batch([
  db.insert(children).values(items.map((i) => ({ parentId: parentRow!.id, ...i }))),
  ...items.map((i) => db.update(other).set({ ... }).where(...)),
] as const);
```

**Reads before writes:** use `Promise.all` for parallel validation selects — they don't need to be in the batch.

### Observability

#### Structured Logging

All logs must be structured JSON. Every line must include:

| Field           | Example                      |
| --------------- | ---------------------------- |
| `timestamp`     | `"2024-01-15T10:30:00.123Z"` |
| `level`         | `"info"`                     |
| `service`       | `"api"`                      |
| `correlationId` | UUID from `X-Correlation-Id` |
| `message`       | Human-readable description   |

Log level rules:

- `DEBUG` — dev tracing only, never enabled in production by default
- `INFO` — normal business events (order created, user logged in)
- `WARN` — recoverable issues, and **ALL 4xx client errors** (400, 404 → INFO/DEBUG, 422 → WARN)
- `ERROR` — server-side failures only (**5xx**). **NEVER use ERROR for 4xx responses.**
- `FATAL` — service cannot continue (missing config, DB unreachable on startup)

Never log PII: passwords, tokens, API keys, credit card numbers, SSNs, full email addresses, or any field ending in `_secret`, `_token`, `_key`, `_password`.

#### Correlation IDs

- Every inbound request: read `X-Correlation-Id` header, generate UUID v4 if absent
- Attach `correlationId` to every log line for the duration of the request
- Every outbound request: forward `X-Correlation-Id` header

### Authentication & Authorization

#### Mechanism: JWT + Refresh Tokens in Neon

- **Access token**: JWT signed with `HS256`, TTL 15 minutes. Verified locally in the Worker via Web Crypto API — no DB hit per request.
- **Refresh token**: Opaque UUID stored in the `sessions` table in Neon. Used only to obtain a new access token.
- **Storage**: httpOnly cookies for both tokens — never accessible from JavaScript.

#### Auth Endpoints

```
POST /api/v1/auth/login    # Validate credentials → set JWT + refresh token cookies
POST /api/v1/auth/refresh  # Validate refresh token in Neon → issue new JWT cookie
POST /api/v1/auth/logout   # Delete session from Neon → clear cookies
GET  /api/v1/auth/me       # Return current user from JWT context
```

#### Session Table (Neon)

```sql
sessions (
  id           uuid PRIMARY KEY,
  user_id      uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash   text NOT NULL UNIQUE,   -- bcrypt hash of the refresh token
  expires_at   timestamptz NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now()
)
```

- Store the **hash** of the refresh token, never the raw value
- Index on `token_hash` for fast lookup
- Delete expired sessions on login (cleanup on write — no background job needed in Workers)

#### Cloudflare Workers Constraint: JWT Signing

Workers have no Node.js `crypto` — use the Web Crypto API:

```typescript
// Use Web Crypto API — NOT jsonwebtoken (Node.js only)
// Recommended library: jose (works with Web Crypto API)
import { SignJWT, jwtVerify } from "jose";
```

Use the `jose` library — it is compatible with the Web Crypto API available in Cloudflare Workers.

#### Hono Auth Middleware

```typescript
// src/middleware/auth.ts
// Verifies JWT from cookie, attaches user to Hono context
// Applied at router level — NEVER per individual route
app.use("/api/v1/protected/*", authMiddleware);
```

- Attach the verified user payload to `c.set("user", payload)` — never re-fetch from DB per request
- Return 401 with standard error shape if token is missing, expired, or invalid
- Return 403 if the user lacks the required role

#### Security Rules

- Validate ALL inputs with Zod at the controller level
- Auth middleware applied at router level — never per-route
- Secrets (`JWT_SECRET`) in `wrangler.toml` secrets — accessed only via `config/index.ts`
- Never log tokens, refresh tokens, or password hashes
- Token cookies: `httpOnly: true`, `secure: true` (production)
- **`sameSite` strategy**: `None` in production (required for cross-origin frontends — e.g. a deployed SPA on a different domain), `Lax` in development. `SameSite=Strict` breaks cookie delivery in any cross-origin setup even with `withCredentials: true`. `SameSite=None` requires `Secure=true` — never set one without the other.
  ```typescript
  const sameSite = isProduction ? "None" : "Lax";
  ```

### Migrations (Drizzle)

- File naming: `YYYYMMDDHHMMSS_description.sql` (UTC timestamp)
- Every migration must have an UP section and a DOWN (rollback) section
- Never DROP COLUMN in production migrations — add nullable column first
- Always test rollback before merging

## Testing

- Unit test services and repositories in isolation (mock the layer below)
- E2E tests use Miniflare local state — see `miniflare-e2e-testing` skill for pitfalls
- Test file location: `src/__tests__/<mirror-path>/` — mirrors the `src/` directory structure (e.g., `src/services/foo.service.ts` → `src/__tests__/services/foo.service.test.ts`)
- Coverage target: services and repositories at 80%+
- Integration tests for all API endpoints
