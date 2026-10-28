# CLAUDE.md — Next.js 15 + SQLite SaaS

This is a **greenfield production SaaS** using Next.js 15 App Router, TypeScript strict mode, React 19, SQLite, and server-side sessions. Follow these instructions without asking which framework, database, route pattern, or naming scheme to use. When an external provider or secret is genuinely missing, implement its interface with a safe local stub rather than inventing credentials.

## Stack and required decisions

- Runtime: Node.js 22 LTS, npm, Next.js 15, React 19, TypeScript strict, ESLint and built-in Next tooling. Pin major versions and commit `package-lock.json` for reproducibility.
- Database: SQLite via `better-sqlite3` for a single durable Node deployment. For edge/serverless/multi-instance hosting, choose Turso/libSQL *before deployment* and replace the driver at the repository boundary; never pretend a local SQLite file is shared between instances.
- App Router uses server components by default. Use client components only when browser state, event handlers, or browser APIs are necessary. This lowers shipped JavaScript and prevents secrets from reaching clients.
- Auth: opaque httpOnly, Secure-in-production, SameSite=Lax session cookie backed by a hashed server-side session token. No homemade JWT authorization or localStorage credentials. Password login, if implemented, uses a maintained password hashing library and rate limiting.
- Validation: Zod or explicit runtime validation at all untrusted boundaries. TypeScript types alone do not validate HTTP inputs.
- Tenant isolation: every tenant-owned table has `organization_id`; every authenticated query scopes by the organization resolved from the server-side session. Never trust a tenant ID supplied by the browser.
- Use UTC ISO timestamps for API/application interchange; persist UTC milliseconds as SQLite INTEGER. Render dates in the user's timezone.

## Directory map (create paths when needed)

```text
src/
  app/
    (public)/page.tsx
    (auth)/login/page.tsx
    (dashboard)/dashboard/page.tsx
    api/health/route.ts
    layout.tsx
    globals.css
  components/
    ui/             # accessible presentational primitives
    features/       # domain-specific UI; no direct DB imports
  server/
    auth/           # sessions, permissions, cookie configuration
    db/
      client.ts     # sole database opening/configuration point
      migrations/   # numbered SQL migration scripts
      migrate.ts
    repositories/   # parameterized SQL, always tenant-scoped
    services/       # business rules, transactions, authorization
  lib/
    validation/     # schemas shared between entry points
    format/         # deterministic formatting helpers
tests/
  unit/
  integration/
public/
.env.example
```

Do not introduce `pages/`, a separate Express server, ORMs, Redux, or a distributed queue just for a simple CRUD feature. Additional abstractions require an actual need documented in the PR.

## Names and imports

- Files for components: `PascalCase.tsx`; React components and types: `PascalCase`; functions and vars: `camelCase`; hooks: `useThing`; SQL identifiers: `snake_case`; route segments: `kebab-case`.
- Use `@/` absolute imports for `src/`; use relative imports inside a tightly coupled module when clearer.
- Prefer named exports except Next.js-required default exports (`page.tsx`, `layout.tsx`, `error.tsx`).
- No `any`, non-null assertions for external data, or suppressed lint errors without a documented reason.

## Database and migrations

1. Keep **only forward, numbered, immutable SQL** migrations such as `0001_initial.sql`, `0002_add_invoices.sql`. Never edit a migration already applied. Schema changes require a new migration and a regression test.
2. `migrate.ts` creates `schema_migrations(version TEXT PRIMARY KEY, applied_at INTEGER NOT NULL)` and applies outstanding files in filename order, each **in a transaction** with its version insert. Stop on any error; never mark a failed migration applied.
3. For `better-sqlite3`, set `PRAGMA foreign_keys = ON`, `journal_mode = WAL`, and `busy_timeout = 5000` on initialization. WAL improves read concurrency but does not eliminate single-writer contention.
4. Use prepared statements with bound parameters for *all* variable values. Identifiers cannot be parameterized: map allowed sort columns through a fixed allowlist. Never concatenate user input into SQL.
5. Prefer SQLite `INTEGER` for booleans (0/1), money as integer minor currency units, and IDs as stable TEXT UUIDs. Store amounts with explicit currency; do not use floating-point for money.
6. Every foreign key and common `(organization_id, status)` filter needs an index if used on growing tables. Add unique constraints for business invariants, not just application checks.
7. All multi-row operations and read-check-write sequences (e.g., create subscription + ledger entry) run in a transaction. Use an idempotency key and unique index for retryable payment/webhook operations.
8. Do not run migrations implicitly on every HTTP request or import. Run `npm run db:migrate` as an explicit deploy step against a backed-up persistent volume; use a separate ephemeral test database.
9. Never commit `*.db`, `*.db-wal`, `*.db-shm`, credentials or customer exports. Back up SQLite consistently (SQLite backup API or safe snapshot), verify restores, and document recovery.
10. Local `better-sqlite3` requires Node runtime: any route using it exports `export const runtime = "nodejs"`. Do not import the DB client into middleware, edge runtime, or client components.

## Request/data flow

- **Read:** Server Component → server service → repository → DB. Fetch only fields needed; no localhost HTTP fetch from server components.
- **Write:** Server Action for first-party form UX or Route Handler for external API → validate → authenticate → authorize tenant/resource → service → repository → transaction → revalidatePath/redirect as appropriate.
- Do not use GET to mutate state. Set expected cache semantics explicitly for user-specific data; never accidentally share private tenant responses through static caching.
- Route Handlers return typed, predictable error bodies and correct HTTP statuses (400 bad input, 401 no auth, 403 forbidden, 404 undisclosed resources, 409 conflict, 429 rate limit, 500 unexpected).
- Keep server-side secrets only in environment variables without `NEXT_PUBLIC_`. Return safe errors to clients; structured server logs may include request IDs but not credentials, session tokens, or PII.

## UI and accessibility

- Prefer semantic HTML and native buttons/forms. Label every input, provide visible focus styles, keyboard actions, disabled/submission states, and error messages linked with `aria-describedby`.
- Treat loading, empty, denied, error, and success states as part of the feature, not optional polish.
- Keep server data and mutation logic out of reusable UI primitives. A `"use client"` boundary should be as deep in the tree as practical.
- Avoid unsafe raw HTML. If external rich text is required, sanitize it using a maintained library on the server.

## Auth and security boundaries

- Require auth and role/resource checks in every service method; hiding UI or relying on middleware is not authorization.
- Hash session tokens in storage, rotate after login/privilege changes, expire them, and invalidate on logout. Set Secure cookies under HTTPS. Use framework-recommended CSRF/origin protections for browser mutations.
- For cross-tenant read/write tests, create two organizations and verify no data leaks in either direction. Deny by default.
- Require provider signature verification for payment webhooks using the original raw body; deduplicate event IDs inside a transaction before any credit is granted.
- Never execute arbitrary shell input, serialize secrets into props, or expose database paths/files through `public/`.

## Commands (run in this order)

```bash
npm ci
npm run lint
npx tsc --noEmit
npm test
npm run build
```

Recommended `package.json` scripts (add when bootstrapping):
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "vitest run",
    "db:migrate": "tsx src/server/db/migrate.ts"
  }
}
```
**Important:** `next lint` was removed from newer Next.js versions; for a Next.js 15 project, prefer `"lint": "eslint ."` when the installed release does not support `next lint`. Configure Vitest and tsx dependencies before running these scripts, rather than claiming missing tools passed. When using native built-in Node tests, use `node --test` instead.

`DATABASE_URL=file:./data/app.db` and `SESSION_SECRET=` belong in `.env.example` as placeholders only. Generate real secrets outside version control. In CI, use a unique temporary database. For tests that mutate schema, provision the database, migrate, and dispose per suite.

## Definition of done

- Show the changed paths, important architecture choice, and any limitations.
- Run lint, TypeScript, unit/integration tests and production build; report actual results, including any skipped commands. Never claim Claude Code runtime validation without launching Claude Code.
- For each DB change, add a migration + rollback/recovery note + test.
- For each authenticated feature, test unauthenticated and other-tenant access.
- For each user-facing form, test invalid input and keyboard interaction.
- Do not deploy or process real customer payments without explicit approval.

## Things we intentionally do not do

- **No SQLite database in ephemeral serverless filesystem**: changes disappear or diverge across instances.
- **No SQL inside React components**: centralize authorization, transaction handling, and testability.
- **No fake security from client-side filtering**: the server always owns access control.
- **No blind schema sync in production**: immutable migrations make changes auditable.
- **No convenience `any` or secrets in `NEXT_PUBLIC_`**: these bypass boundaries.
- **No promises about unexecuted tests**: evidence is required for a production change.
