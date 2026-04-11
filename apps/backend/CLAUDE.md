# Backend — StockFlow API

REST API deployed to Cloudflare Workers using Hono.js, Drizzle ORM, Neon PostgreSQL, and Zod.

## Stack

| Layer       | Technology                               |
| ----------- | ---------------------------------------- |
| Runtime     | Cloudflare Workers (edge)                |
| Framework   | Hono.js                                  |
| Database    | Neon PostgreSQL (serverless HTTP driver) |
| ORM         | Drizzle ORM                              |
| Validation  | Zod v4                                   |
| Cache       | In-memory API cache at the service layer |
| Local dev   | Wrangler + Miniflare                     |
| Testing     | Vitest + Miniflare E2E                   |

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

### Zod v4 Schemas (models/)

Zod v4 breaking changes — always use the new API:

```typescript
// ❌ Zod 3 (OLD — do not use)
z.string().email()
z.string().uuid()
z.string().url()
z.string().nonempty()
z.object({ name: z.string() }).required_error("Required")

// ✅ Zod 4 (NEW — always use this)
z.email()
z.uuid()
z.url()
z.string().min(1)
z.object({ name: z.string() }, { error: "Required" })
```

- Use `z.infer<typeof Schema>` for TypeScript types — never define types separately
- Validate at the controller level — never trust raw request body in services or repositories
- Error messages use `{ error: "..." }` not `{ message: "..." }`

### API Cache (service layer)

- Cache sits inside service methods — NOT in controllers or repositories
- Cache key pattern: `resource:id` or `resource:list:hash(filters)`
- Always invalidate on mutations (create, update, delete)
- Always use TTL — never cache indefinitely

### Cloudflare Workers Constraints

- No Node.js APIs — use Web Standard APIs only (fetch, crypto, Request, Response)
- No long-running tasks — Workers have CPU time limits
- Secrets via `wrangler.toml` and `c.env` — accessed only through `config/index.ts`
- Neon uses HTTP driver (`@neondatabase/serverless`) — not standard TCP pg connection

### Database Patterns

- All SQL uses Drizzle ORM — never raw string interpolation
- Any operation writing to 2+ tables must use a transaction
- Services coordinate transactions; repositories accept a connection/transaction parameter
- N+1 prevention: when fetching a list with related data, always batch — never fetch per item
- Never return raw DB errors to the client — map all DB errors to domain errors in repositories

### Observability

#### Structured Logging

All logs must be structured JSON. Every line must include:

| Field           | Example                          |
| --------------- | -------------------------------- |
| `timestamp`     | `"2024-01-15T10:30:00.123Z"`     |
| `level`         | `"info"`                         |
| `service`       | `"api"`                          |
| `correlationId` | UUID from `X-Correlation-Id`     |
| `message`       | Human-readable description       |

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
- Token cookies: `httpOnly: true`, `secure: true` (production), `sameSite: Strict`

### Migrations (Drizzle)

- File naming: `YYYYMMDDHHMMSS_description.sql` (UTC timestamp)
- Every migration must have an UP section and a DOWN (rollback) section
- Never DROP COLUMN in production migrations — add nullable column first
- Always test rollback before merging

## Testing

- Unit test services and repositories in isolation (mock the layer below)
- E2E tests use Miniflare local state — see `miniflare-e2e-testing` skill for pitfalls
- Test file location: co-located with source (`*.test.ts`)
- Coverage target: services and repositories at 80%+
- Integration tests for all API endpoints
