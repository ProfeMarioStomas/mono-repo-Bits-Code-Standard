# Frontend — StockFlow

React SPA built with Vite, TanStack Query, TanStack Form, Tailwind CSS, and Axios.

## Stack

| Layer        | Technology            |
| ------------ | --------------------- |
| Bundler      | Vite                  |
| UI Framework | React 19 (TypeScript) |
| Server State | TanStack Query v5     |
| Forms        | TanStack Form v1      |
| HTTP Client  | Axios                 |
| Styling      | Tailwind CSS v4       |
| Validation   | Zod v4                |

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
    retry: false, // Don't retry on 401
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
- **Always exclude auth-checking endpoints from the refresh retry** — `/auth/me` returns 401 when the user is not logged in (expected, not an expired token). If the interceptor tries to refresh on that 401, it creates an infinite loop: `LoginPage` calls `useCurrentUser()` → 401 → interceptor retries refresh → 401 → hard redirect to `/login` → `LoginPage` mounts again → loop. The exclusion list must include at minimum `/auth/refresh` and `/auth/me`.

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
- Pass the Zod schema directly via Standard Schema — no adapter package needed:
  ```typescript
  const form = useForm({
    validators: { onSubmit: z.object({ email: z.email() }) },
  });
  ```
- Never manage form state manually with `useState` — TanStack Form handles it
- Submit handler calls the service function, then triggers TanStack Query mutation

#### Numeric fields — coercion before API call (REQUIRED)

TanStack Form's `onSubmit` receives the **raw form state**, not the Zod-parsed output.
HTML `<input type="number">` always returns a **string** (e.g. `"19.99"`), even when
the field type is declared as `number`. The `z.coerce.number()` in the schema coerces
the value during validation so no field error is shown, but the raw string is what gets
passed to `onSubmit` — and then sent to the backend, where `z.number()` (no coerce)
fails with `"Price must be a number"`.

**Always parse through the Zod schema inside `onSubmit` before calling the service:**

```typescript
onSubmit: async ({ value }) => {
  // value.price is "19.99" (string) — coerce it before sending
  const coerced = mySchema.parse(value);
  await myService.create(coerced); // ✅ price is now 19.99 (number)
},
```

#### Optional numeric fields — empty input must map to `undefined`

`z.coerce.number()` converts `""` to `0`. If the field also has `.positive()`, this
causes a validation error when the user leaves the field empty. Fix it in two places:

1. **`defaultValues`**: use `undefined`, not `""`

   ```typescript
   defaultValues: {
     costPrice: undefined as number | undefined,
   }
   ```

2. **`onChange`**: convert empty string to `undefined`
   ```typescript
   onChange={(e) =>
     field.handleChange(
       e.target.value === "" ? undefined : (e.target.value as unknown as number),
     )
   }
   ```

#### Server error display — always show `details[]`

The standard error shape is `{ error: { code, message, details[] } }`. Never display
only `error.message` — always render `error.details` as a list so the user knows which
fields failed:

```typescript
// State type
type ServerError = {
  message: string;
  details?: { field: string; message: string }[];
};

// In catch block
const error = axiosError.response?.data?.error;
setServerError({
  message: error?.message ?? "An unexpected error occurred.",
  details: error?.details?.length ? error.details : undefined,
});
```

```tsx
{
  /* In JSX */
}
{
  serverError && (
    <div role="alert" className="...">
      <p>{serverError.message}</p>
      {serverError.details && (
        <ul className="mt-1 list-inside list-disc">
          {serverError.details.map((d, i) => (
            <li key={i}>
              <span className="font-medium">{d.field}</span>: {d.message}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

#### Dynamic array fields — `mode="array"` (REQUIRED pattern)

Use TanStack Form's `mode="array"` for forms with a variable number of rows (e.g.
inventory receipt items, sale line items). Never manage the array with `useState`.

```tsx
<form.Field name="items" mode="array">
  {(field) => (
    <div className="flex flex-col gap-2">
      {field.state.value.map((_, i) => (
        <div key={i} className="flex items-end gap-2">
          {/* Nested subfields use bracket notation */}
          <form.Field name={`items[${i}].productId`}>
            {(sub) => <select value={sub.state.value} onChange={...} />}
          </form.Field>

          <form.Field name={`items[${i}].quantity`}>
            {(sub) => <input type="number" value={sub.state.value} onChange={...} />}
          </form.Field>

          <button onClick={() => field.removeValue(i)}>Remove</button>
        </div>
      ))}

      {/* Array-level validation error (e.g. min(1)) */}
      {field.state.meta.errors[0] && (
        <p className="text-xs text-[var(--color-error-text)]">
          {field.state.meta.errors[0].message}
        </p>
      )}

      <button onClick={() => field.pushValue({ productId: "", quantity: "" as unknown as number })}>
        Add Item
      </button>
    </div>
  )}
</form.Field>
```

Key rules:

- **`field.pushValue(item)`** — adds a row at the end
- **`field.removeValue(index)`** — removes by index
- **Nested path syntax**: `items[${i}].fieldName` — always square brackets, never dot notation
- **Coercion still applies**: call `schema.parse(value)` in `onSubmit` — coerce applies to
  array item fields too (see "Numeric fields — coercion before API call" above)
- **Array-level errors** surface on `field.state.meta.errors[0]`, not on individual subfields
- **Auto-filling sibling fields**: use `(form as any).setFieldValue(path, value)` when
  selecting a product should pre-populate a price field in the same row

#### Cross-resource cache invalidation

When a mutation in one resource affects the state of another (e.g. creating a sale or an
inventory receipt changes product stock levels), **always invalidate all affected query keys**,
not just the primary resource.

```typescript
// ✅ Invalidate both the sale and the products whose stock changed
await queryClient.invalidateQueries({ queryKey: ["sales"] });
await queryClient.invalidateQueries({ queryKey: ["products"] });

// Same for inventory receipts (create or void)
await queryClient.invalidateQueries({ queryKey: ["inventory-receipts"] });
await queryClient.invalidateQueries({ queryKey: ["products"] });
```

Missing a secondary invalidation leaves the UI showing stale stock numbers until the
next manual refresh.

#### Role-based field access

Use `useCurrentUser()` to read the authenticated user's role and conditionally lock
fields. Never pass roles as props — always read from the query.

```typescript
const { data: currentUser } = useCurrentUser();
const isAdmin = currentUser?.role === "admin";
```

Apply `readOnly` (not `disabled`) so the value is still visible and tracked in form state:

```tsx
<input
  type="number"
  readOnly={!isAdmin}
  className="... read-only:cursor-not-allowed read-only:opacity-60"
  value={...}
  onChange={...}
/>
```

`CurrentUser` shape returned by `GET /auth/me`:

```typescript
type CurrentUser = {
  id: string;
  name: string;
  email: string;
  role: "admin" | "seller";
};
```

### Zod v4 Schemas (models/)

Zod v4 breaking changes — always use the new API:

```typescript
// ❌ Zod 3 (OLD — do not use)
z.string().email();
z.string().uuid();
z.string().min(1, { message: "Required" });

// ✅ Zod 4 (NEW — always use this)
z.email();
z.uuid();
z.string().min(1, { error: "Required" });
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

### Page vs Modal responsibility

- **Pages show data** — tables, reports, dashboards, read-only views
- **Modals contain forms** — create, edit, and delete confirmations are ALWAYS in a modal, never on a separate page
- Every CRUD feature follows this pattern: page renders the table + action buttons → button opens a modal → modal contains the TanStack Form → on submit, modal closes and the query is invalidated
- Delete actions use a confirmation modal (never a browser `confirm()` or inline action without confirmation)

### Edit modal preloading — REQUIRED pattern

Edit modals **must** always receive the entity as a prop and use its fields in `defaultValues`. The parent **must** set `key={entity.id}` on the modal component:

```tsx
// ✅ CORRECT — key forces a fresh mount per entity, so useForm always
// initializes with the right defaultValues. Never use useEffect + form.reset()
// to sync values; that causes a flash of empty fields.
{modal?.type === "edit" && (
  <EditFooModal key={modal.foo.id} foo={modal.foo} open onClose={...} />
)}

// ❌ WRONG — no key means React may reuse the component instance when
// switching between entities, leaving stale form values.
{modal?.type === "edit" && (
  <EditFooModal foo={modal.foo} open onClose={...} />
)}
```

**Why `key` and not `useEffect + form.reset()`:** TanStack Form initializes `defaultValues` on mount. Calling `form.reset()` in a `useEffect` runs after the first paint and clears then reapplies values, causing a visible flash of empty fields. The `key` approach remounts the component cleanly so `useForm` gets the correct values from the start.

### R2 Image Upload

**Public base URL**: `https://pub-f0bcf28b115849ffbbb6ac15fb70a6c2.r2.dev`

#### Upload-then-save flow (Option A)

When a form includes an image field, upload the file first, then pass the returned key
to the create/update call. Never send the `File` object directly to the entity endpoint.

```typescript
// In onSubmit:
let imageKey: string | undefined;
if (imageFile) {
  const uploaded = await myService.uploadImage(imageFile);
  imageKey = uploaded.key;
}
await myService.createEntity({ ...coerced, imageKey });
```

#### Service method

```typescript
uploadImage: (file: File) => {
  const formData = new FormData();
  formData.append("file", file);
  return api
    .post<{ key: string; url: string }>("/resource/images", formData)
    .then((r) => r.data);
},
```

Axios sets `Content-Type: multipart/form-data` automatically when the body is a `FormData`.
Never set it manually — doing so omits the boundary parameter and breaks the request.

#### Displaying images

Construct the full URL from the public base URL and the stored key:

```typescript
const R2_BASE_URL = "https://pub-f0bcf28b115849ffbbb6ac15fb70a6c2.r2.dev";
const imageUrl = product.imageKey ? `${R2_BASE_URL}/${product.imageKey}` : null;
```

#### File input in modals

- Track the selected file with `useState<File | null>(null)` (separate from form state)
- Generate a local object URL with `URL.createObjectURL(file)` for the preview
- In edit modals: show the existing image from R2 when no new file is selected, replace with the local preview when the user picks a new file
- Reset `imageFile` and `imagePreview` state when the modal closes (in `handleClose`)
- Accepted formats: `image/jpeg,image/png,image/webp` — validate on the backend too

### Error Handling

- TanStack Query `error` state for async errors — display in UI, not console
- Form errors via TanStack Form field validation — display inline
- Network/auth errors handled globally in Axios interceptor (e.g., redirect on 401)

## UI/UX Design

Always invoke the `ui-ux-pro-max` skill before implementing any new UI component, page, or layout.
This skill provides design guidance, color palettes, typography, spacing, and component patterns
tailored to the product type and stack.

## Testing

- Component tests with `@testing-library/react` + Vitest
- Mock TanStack Query with `QueryClientProvider` wrapping test render
- Mock Axios with `msw` (Mock Service Worker) — never mock modules directly
- Test file location: `src/__tests__/<mirror-path>/` — mirrors the `src/` directory structure (e.g., `src/services/foo.service.ts` → `src/__tests__/services/foo.service.test.tsx`)
- E2E tests for critical user flows
