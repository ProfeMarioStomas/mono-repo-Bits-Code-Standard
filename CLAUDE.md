# StockFlow

Monorepo with backend API and frontend SPA as separate applications.

## Structure

```
apps/
  backend/    # Cloudflare Workers API — Hono.js, Drizzle ORM, Neon PostgreSQL, Zod
  frontend/   # React SPA — Vite, TanStack Query, TanStack Form, Tailwind CSS, Axios
```

## Package Manager

pnpm workspaces

## Global Commands

```bash
pnpm install              # Install all dependencies
pnpm -F backend dev       # Start backend dev server (Wrangler)
pnpm -F frontend dev      # Start frontend dev server (Vite)
pnpm -F backend test      # Run backend tests
pnpm -F frontend test     # Run frontend tests
pnpm -F backend lint      # Lint backend
pnpm -F frontend lint     # Lint frontend
pnpm -F backend format    # Format backend
pnpm -F frontend format   # Format frontend
```

## Conventions

- TypeScript throughout — strict mode enabled in all packages
- All code, comments, and documentation in English
- Conventional commits format (feat, fix, chore, docs, refactor, test)
- Zod schemas are the source of truth for data validation at every boundary
- No `any` types — use `unknown` with type guards when type is genuinely unknown
- No unused imports, variables, or dead code

## Branch Strategy

- `main` — production-ready code
- `develop` — integration branch
- `feature/*` — new features
- `bugfix/*` — bug fixes
- `hotfix/*` — urgent production fixes

## Standard Error Response Shape

Every error from every endpoint must use this exact shape — no exceptions:

```json
{
  "error": {
    "code": "SCREAMING_SNAKE_CASE_CODE",
    "message": "Human-readable description",
    "details": []
  }
}
```

- `code` is a stable machine-readable string in SCREAMING_SNAKE_CASE (never change once published)
- `details` is always an array — use for field-level validation errors, empty array otherwise
- Never expose stack traces, SQL, or internal paths in error responses
- **NEVER return 200 with `{ success: false }`** — always use 4xx/5xx status codes

## Shared Patterns

- DTOs and shared types live in `apps/backend/src/models/`
- All dates are ISO 8601 strings over the wire

## Pre-commit Checklist

Before any commit: format → lint → test → CI validation.
Never skip. Never assume. Read actual output.
