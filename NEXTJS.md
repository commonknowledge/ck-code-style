@AGENTS.md

# Project Guidelines

## Stack

- **Framework**: Next.js (App Router)
- **Language**: TypeScript (strict mode)
- **Linting/Formatting**: Biome (strict)
- **Database**: Kysely + PostgreSQL, unless otherwise specified
- **Schemas/Validation**: Zod
- **Environment variables**: dotenv, t3-env
- **Component library**: Shadcn
- **Testing**: Vitest

### Project bootstrap

When creating a new project with `create-next-app`, enable the `src/` directory option. The `app/` router and all module directories (`components/`, `models/`, `server/`, etc.) must live under `src/`.

### Stack options

- **Client/Server communication**: Default to Server Actions, server provided props and `revalidatePath`
to trigger re-renders. Introduce tRPC when you have many endpoints, need a typed client consumed across multiple surfaces, or need fine-grained cache control via react-query.
- **Client state**: Default to React state and context. Introduce Jotai when state needs to be shared across 
component trees that don't share a convenient ancestor, or when atom composition would meaningfully reduce 
complexity.

### Library discipline

Do not install a library unless the user has explicitly requested it. If a task would be
significantly cleaner or simpler with a specific library, suggest it to the user and wait
for confirmation before proceeding.

---

## TypeScript

- Strict mode is enabled. Never relax it.
- Avoid the `as` operator. Use type guards, Zod parsing, or a well-typed helper instead.
- Prefer `unknown` over `any`. If `any` is genuinely required, add a comment explaining why.
- Prefer `interface` for object shapes that may be extended; `type` for unions, intersections, and aliases.

---

## Environment Variables

Use `@t3-oss/env-nextjs` to declare and validate all variables.
Call `dotenv.config()`, at the top of the main env/config module
so `.env` files are loaded before any env vars are read.
Do not access `process.env` directly anywhere else in the codebase.

Set `DOTENV_CONFIG_QUIET=true` in all `.env` files.

**Test overrides:** Vitest config files should load `.env.test` explicitly via
`dotenv.config({ path: '.env.test' })` and pass the parsed values into Vitest's
`test.env`. This means `.env.test` values are injected into `process.env` before any test
module runs, overriding `.env`.

Keep all `.env*` files sorted alphabetically by key.

---

## Code Style

### Naming

| Thing | Convention |
|---|---|
| Files (non-component) | `camelCase.ts` |
| Component files | `TitleCase.tsx` |
| Class files | `TitleCase.ts` |
| Types and interfaces | `TitleCase` |
| Variables, functions, methods | `camelCase` |
| Constants | `UPPER_SNAKE_CASE` |

### React

- Components must be the `default` export from a file matching their name.
- Complex components that need their own directory: place the component in a file with the same name as the directory, e.g. `Example/Example.tsx`. Co-locate sub-components, styles, and tests within that directory.
- Hooks that call `useEffect` must end with `Effect`, e.g. `useExampleEffect`.
- Prefer named functions over arrow functions at the module level for better stack traces.
- When a `useEffect` depends on a function defined inside the component, stabilise it with `useCallback` rather than restructuring the code to avoid the dependency. Only extract a function to module scope if it genuinely has no component dependencies.

### Biome

Biome is the single source of truth for formatting and linting. Do not add ESLint or Prettier.
All rules are set to strict; do not suppress warnings without a comment explaining why.

When running Biome, always run in `--write` mode to automatically fix simple issues and reduce
token usage.

---

## Module Naming Conventions

Modules should be grouped by the following conventions. Not every project will use every
group, and additional groupings are permitted where a clear purpose exists.

| Directory | Contains |
|---|---|
| `components/` | React components |
| `constants/` | Constant values (`UPPER_SNAKE_CASE` exports) |
| `hooks/` | Custom React hooks |
| `models/` | Zod schemas representing application domain models |
| `providers/` | React context providers |
| `server/` | Server-only modules — must never be imported client-side |
| `services/` | Modules that are async, have side effects, or cross a system boundary (e.g. database access, external APIs, file I/O) |
| `styles/` | Global styles and Tailwind config helpers |
| `utils/` | Pure, synchronous helper functions |

Other well-understood groupings: `adaptors/`, `api/`, `commands/`, `jobs/`, `repositories/`, `routes/`.

Avoid `helpers/` — there is almost always a more descriptive category.

---

## State Management

### Ephemeral UI state → React state or Jotai atoms

Use `useState` and Jotai atoms for transient, UI-specific state that the server does not
need to know about. Keep state as local as possible; lift it only when genuinely shared.
Introduce Jotai when state needs to be shared across component trees that don't share a
convenient ancestor, or when atom composition would meaningfully reduce complexity.

### Client/server communication → Server Actions or tRPC

**Default: Server Actions.** Mutations call a Server Action directly; on success the action
calls `revalidatePath` (or `redirect`) to trigger a re-render with fresh server-provided
props. No client state is needed to store server data — the server re-renders with the
updated values.

**When tRPC is in use:** No data for which the server is the source of truth should be
copied into local React or Jotai state unless absolutely necessary. Instead, server-provided
props or the react-query cache should be used. Mutations should write to the query cache to
trigger re-renders.

Introduce tRPC (see [tRPC](#trpc) section) when Server Actions are insufficient — for
example when you have many endpoints, need a typed client consumed across multiple surfaces,
or need fine-grained cache control via react-query.

---

## tRPC

> Use tRPC when Server Actions are not sufficient — see [Stack options](#stack-options) for
> the decision criteria. When the project uses Server Actions as its primary communication
> layer, this section applies only to the subset of endpoints that require tRPC.

Routers live in `server/routers/`. Each domain area gets its own router, composed into
the root `appRouter` in `server/routers/_app.ts`.

```
server/
  routers/
    _app.ts          # root router — merges all sub-routers
    public/
        ...
```

### Procedure conventions

Procedure types should be defined:

| Export | Accessible by |
|---|---|
| `publicProcedure` | Anyone (no auth check) |
| `privateProcedure` | Authorized users |
| `...Procedure` | Other reusable procedures |

- All inputs must be validated with a Zod schema. Reuse schemas from `models/` where they apply.
- Return plain serialisable objects. Do not return class instances.
- Use `TRPCError` with appropriate codes rather than throwing generic errors:

| Situation | Code |
|---|---|
| Unauthenticated | `UNAUTHORIZED` |
| Authenticated but forbidden | `FORBIDDEN` |
| Resource not found | `NOT_FOUND` |
| Bad input (beyond Zod) | `BAD_REQUEST` |
| Unexpected failure | `INTERNAL_SERVER_ERROR` |

### When to use `app/api/` route handlers

tRPC is the default API layer when tRPC is in use. Use Next.js route handlers
(`app/api/`) only when tRPC is inappropriate — for example:

- Webhook endpoints that receive requests from external services.
- File upload/download endpoints that deal with streams or binary data.
- Endpoints that must conform to a fixed external contract (OAuth callbacks, payment
  provider redirects, etc.).
- Endpoints that serve raw HTML or non-JSON content.

Route handlers should still validate input with Zod and return appropriate HTTP status
codes. Keep business logic in `server/` services; do not inline complex logic in the
route file.

---

## Database

The database client should be a `Kysely<Database>` instance, unless otherwise specified.

@KYSELY.md

---

## Zod & Application Models

Domain models are Zod schemas in `models/`. Derive TypeScript types from them with `z.infer`.

```ts
// models/User.ts
import { z } from 'zod'

export const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1),
})

export type User = z.infer<typeof UserSchema>
```

- Parse external data at the boundary (API responses, env vars, form inputs). Do not use
  `.parse` deep inside business logic — parse early, pass typed values inward.
- Prefer `.parse` (throws) in server contexts, `.safeParse` (returns result) in UI
  contexts where you need to show validation feedback.

---

## Proxy (`proxy.ts`)

Next.js proxy lives in `proxy.ts` at the project root (Next.js v16 renamed `middleware.ts`
to `proxy.ts`). It runs on the Node.js runtime before every matched request.

### Allowed uses

- Path redirects and rewrites.
- Setting or rewriting request headers (e.g. `x-forwarded-*`, locale hints).
- Geographic or locale-based routing.

### Prohibited uses

- **No authentication or authorisation logic.** Do not read, validate, or enforce sessions,
  tokens, or roles in proxy. Auth checks must happen in layouts or server components where
  the full server runtime is available and the auth context can be properly resolved.
- No database or service calls — keep proxy fast and stateless.

---

## Error Handling

### General principles

- Handle errors at the boundary where you can do something meaningful with them.
- Let unexpected errors propagate to the nearest error boundary or tRPC error handler
  rather than silently swallowing them.
- Log unexpected errors server-side. Do not log sensitive data.

### Server (Server Actions and tRPC procedures)

**Server Actions** (default communication layer):
- Validate inputs with Zod (`.parse` throws on failure).
- On success, call `revalidatePath` or `redirect` — do not return raw data to the client.
- On known failure, return a safe user-facing message. Do not return raw error objects or
  expose internal details.
- Log unexpected errors server-side before returning a generic fallback message.

**tRPC procedures** (when tRPC is in use):
- Throw `TRPCError` with an appropriate code for all known failure cases.
- Do not expose internal error messages or stack traces to the client. Use a safe
  user-facing message and log the original error server-side.

### Client (React)

- Use React error boundaries to catch rendering errors. Place them around route segments
  and major UI regions.
- For mutation errors surfaced by Server Actions, display feedback inline. Surface a
  user-facing message for known failures; use a generic fallback for unexpected ones.
- Never expose raw error objects or stack traces in the UI.

### Form validation

- Use Zod schemas for client-side validation.
- Always re-validate server-side in the Server Action or tRPC procedure, even if the
  client validates first.

---

## English Dialect

- **Code** (variable names, file names, comments in code): American spellings.
- **User-facing text, documentation, and non-inline comments**: British spellings.

---

## Testing

### Integration — Vitest

Create tests for all new features unless explicitly told not to.

- Test files live alongside their subject: `utils/formatDate.ts` → `utils/formatDate.test.ts`.
- For components, place tests in the component's directory.
- Test pure functions exhaustively. For components, test behaviour not implementation.
- Use `vi.mock` to mock external services and side effects in unit tests. Do not make real
  network calls in unit tests.

**Testing Server Actions:**
- Test the happy path (successful mutation, correct `revalidatePath`/`redirect` call) and
  the failure paths (Zod validation errors, known domain errors).
- Mock `revalidatePath` and `redirect` from `next/navigation` with `vi.mock` in unit tests.
- Do not assert on raw return values — assert on the mocked side effects (`revalidatePath`
  called with the expected path, `redirect` called with the expected URL).

**Testing tRPC procedures** (when tRPC is in use):
- Call procedures via a test caller created with `appRouter.createCaller(ctx)`.
- Assert that `TRPCError` is thrown with the correct code for all known failure cases.

Integration tests:

- Must not use mocks — use `.env.test` environment variables instead.
- Should provision and destroy external dependencies around the test suite (e.g. run
  migrations against the test database in `beforeAll`, clean up in `afterAll`).
- Live in the `tests/integration/` directory at the project root.

<!-- BEGIN:nextjs-agent-rules -->
# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.
<!-- END:nextjs-agent-rules -->
