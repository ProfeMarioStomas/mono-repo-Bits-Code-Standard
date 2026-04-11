# Frontend — StockFlow

React SPA built with Vite, TanStack Query, TanStack Form, Tailwind CSS, and Axios.

## Stack

| Layer           | Technology                   |
| --------------- | ---------------------------- |
| Bundler         | Vite                         |
| UI Framework    | React 19 (TypeScript)        |
| Server State    | TanStack Query v5            |
| Forms           | TanStack Form v1             |
| HTTP Client     | Axios                        |
| Styling         | Tailwind CSS v4              |
| Validation      | Zod v4                       |

## Project Structure

```
src/
  main.tsx              # App entry point
  App.tsx               # Root component — providers, routing
  pages/                # Page-level components (one per route)
  components/
    common/             # Generic, reusable UI components (Button, Input, Modal, etc.)
    features/           # Domain-specific components (co-located with their feature)
    layouts/            # Layout components (sidebar, header, shells)
  hooks/                # Custom React hooks
  services/
    api.ts              # Axios instance — base URL, interceptors, auth headers
    *.service.ts        # Domain-specific API call functions (used by TanStack Query)
  store/                # Client-only state (Zustand if needed)
  models/               # Zod schemas for form validation and API response parsing
  types/                # TypeScript interfaces and shared types
  lib/
    queryClient.ts      # TanStack Query client configuration
    cn.ts               # cn() utility (clsx + tailwind-merge)
  styles/
    globals.css         # Tailwind base imports and CSS custom properties
```

## Commands

```bash
pnpm dev        # vite — starts dev server
pnpm build      # vite build — production build
pnpm preview    # vite preview — preview production build locally
pnpm test       # vitest run
pnpm lint       # eslint
pnpm format     # prettier --write
```

## Architecture Rules

### Server State (TanStack Query)

- All server data lives in TanStack Query — never duplicate in local state
- Query key pattern: `['resource']` for lists, `['resource', id]` for single items
- API calls are plain async functions in `src/services/*.service.ts`
- Mutations always invalidate relevant query keys on success
- **Never use `useEffect` to fetch data** — that's what `useQuery` is for

### Authentication & Authorization

#### Mechanism: JWT + Refresh Tokens (httpOnly cookies)

Both the access token (JWT, 15 min) and the refresh token are set as httpOnly cookies by the backend. The frontend never reads or stores tokens directly — the browser sends them automatically on every request.

#### Auth State

Use a `useQuery` hook to fetch the current user — this is the source of truth for auth state:

```typescript
// hooks/useCurrentUser.ts
export function useCurrentUser() {
  return useQuery({
    queryKey: ["auth", "me"],
    queryFn: () => authService.me(),
    retry: false,           // Don't retry on 401
    staleTime: 5 * 60_000, // 5 min — avoid hammering /auth/me
  });
}
```

#### Protected Routes

Wrap authenticated pages in a guard component that redirects to `/login` on 401:

```typescript
// components/common/ProtectedRoute.tsx
function ProtectedRoute({ children }: { children: ReactNode }) {
  const { data: user, isLoading, isError } = useCurrentUser();

  if (isLoading) return <PageSpinner />;
  if (isError) return <Navigate to="/login" replace />;

  return children;
}
```

#### Token Refresh Flow (Axios interceptor)

Handle token expiration transparently in `src/services/api.ts`:

```
Request → 401 received
  → Call POST /api/v1/auth/refresh (browser sends refresh cookie automatically)
  → If refresh succeeds → retry original request once
  → If refresh fails → clear query cache → redirect to /login
```

- Use a flag to avoid infinite refresh loops (only retry once per request)
- Never implement this logic in individual service functions — only in the interceptor

#### Rules

- Never store JWT or tokens in `localStorage`, `sessionStorage`, or React state — cookies only
- Never read cookie values from JavaScript — they are httpOnly
- On logout: call `POST /api/v1/auth/logout` then invalidate all TanStack Query cache (`queryClient.clear()`)

### HTTP Client (Axios)

- Single Axios instance in `src/services/api.ts` — configure interceptors there
- Auth handled automatically via httpOnly cookies — no manual token attachment needed
- Response interceptor handles 401: attempt refresh → retry → redirect to `/login`
- API errors normalized to standard error shape `{ error: { code, message, details } }`
- Never call `axios.get()` directly in components — always through service functions

### Forms (TanStack Form + Zod v4)

- All form validation via Zod schemas defined in `src/models/`
- Use `@tanstack/zod-form-adapter` for integration
- Never manage form state manually with `useState` — TanStack Form handles it
- Submit handler calls the service function, then triggers TanStack Query mutation

### Zod v4 Schemas (models/)

Zod v4 breaking changes — always use the new API:

```typescript
// ❌ Zod 3 (OLD — do not use)
z.string().email()
z.string().uuid()
z.string().min(1, { message: "Required" })

// ✅ Zod 4 (NEW — always use this)
z.email()
z.uuid()
z.string().min(1, { error: "Required" })
```

- Use `z.infer<typeof Schema>` for TypeScript types — never define types separately
- Parse API responses with Zod — never trust raw API shape in component code

### Styling (Tailwind CSS v4)

#### cn() Utility — Required Pattern

```typescript
// src/lib/cn.ts
import { clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

Usage rules:
```typescript
// ✅ Use cn() for conditional classes
<div className={cn("base-class", isActive && "active-class")} />

// ✅ Use cn() for potential conflicts
<button className={cn("px-4 py-2", className)} />

// ❌ Don't use cn() for static classes — just use className directly
<div className="flex items-center gap-2" />
```

#### Critical Rules

```typescript
// ❌ NEVER: var() in className
<div className="bg-[var(--color-primary)]" />

// ✅ ALWAYS: Tailwind semantic classes
<div className="bg-primary" />

// ❌ NEVER: hex colors
<p className="text-[#ffffff]" />

// ✅ ALWAYS: Tailwind color classes
<p className="text-white" />
```

- Dynamic numeric values use `style` prop: `<div style={{ width: `${pct}%` }} />`
- When 3rd-party libraries don't accept `className` (e.g., chart libraries), use CSS custom properties as constants
- Responsive design mobile-first: `base → sm → md → lg → xl`

### Components

- Functional components with TypeScript props interface defined above the component
- Props interfaces named `[ComponentName]Props`
- No prop drilling beyond 2 levels — lift to TanStack Query or context
- Prefer composition over configuration — use children and render props over boolean flags
- Co-locate feature components with their page, not in a global components folder

### Error Handling

- TanStack Query `error` state for async errors — display in UI, not console
- Form errors via TanStack Form field validation — display inline
- Network/auth errors handled globally in Axios interceptor (e.g., redirect on 401)

## Testing

- Component tests with `@testing-library/react` + Vitest
- Mock TanStack Query with `QueryClientProvider` wrapping test render
- Mock Axios with `msw` (Mock Service Worker) — never mock modules directly
- Test file location: co-located with source (`*.test.tsx`)
- E2E tests for critical user flows
