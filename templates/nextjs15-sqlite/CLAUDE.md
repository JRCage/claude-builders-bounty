# Project instructions — Next.js 15 App Router + SQLite SaaS

This is the operating contract for a **greenfield** Next.js 15 App Router, TypeScript, React, SQLite SaaS. Prefer a small, auditable server-rendered application to a premature microservice architecture. If the repo differs, inspect package.json, migrations and the current code before changing it; never invent project-specific scripts.

## Stack and versions

- Next.js **15** with App Router and React **19**, TypeScript in strict mode; Node.js **20+**. Pin the package manager and lockfile in the actual project.
- **Choose one** SQLite adapter at setup: `better-sqlite3` for a single persistent Node server or **Turso/libSQL** for hosted/edge environments. Do not silently mix the adapters in a deployment.
- TypeScript server runtime for DB routes: `export const runtime = "nodejs"` with better-sqlite3, since native SQLite bindings cannot run in Edge.
- Use Drizzle ORM plus drizzle-kit **if included** in package.json; otherwise use parameterized SQL and numbered SQL migrations. Never introduce a second migration authority. Existing schema and migration history are the source of truth.
- Treat authentication, email, payments, storage and telemetry as **not configured** until the actual repo declares them. Never fabricate environment variables, providers or production deployment settings.

## Repository layout

```text
src/
  app/
    (marketing)/page.tsx           # public content
    (app)/dashboard/page.tsx      # authenticated UI; verify session on server
    api/health/route.ts           # minimal GET health check, no secrets
    layout.tsx
  components/
    ui/                           # reusable presentation; no DB access
    features/                     # feature-specific UI
  lib/
    db/
      client.ts                    # singleton server-only DB connection
      queries/                     # parameterized reads/writes
      schema.ts                    # canonical schema if ORM is used
    auth/                          # session validation when configured
    validation/                    # request input parsers
  actions/                         # server mutations; enforce auth each call
  types/
migrations/                        # numbered, committed, append-only
tests/
  unit/
  integration/
.env.example                       # placeholders only, no real credentials
```

Use `kebab-case` for file names and routes, `PascalCase` for React components/types, `camelCase` for variables/functions, `SCREAMING_SNAKE_CASE` for environment variables. Name DB tables and columns `snake_case`. Keep a feature's controller and validation close to the domain, rather than creating giant utility modules.

## Commands (check package.json before running)

On a fresh starter, establish and commit scripts for `dev`, `build`, `lint`, `typecheck`, `test`, `db:migrate` and `db:check`; `typecheck` can be `tsc --noEmit`. Do not claim a script exists before inspecting package.json.

```sh
npm ci
npm run dev
npm run lint && npm run typecheck && npm test && npm run build
```

Use the project's lockfile manager instead of npm when different. Never run `npm audit fix --force`, upgrade major versions, or apply migrations as a side effect of building.

## App Router boundaries

1. Server Components by default. Add `"use client"` **only** for event handlers, browser state and browser-only APIs. This reduces shipped JS and prevents importing secrets into client bundles.
2. Server-only modules that hold secrets, DB connections or admin APIs must import `server-only`; never pass secret values to client props or `NEXT_PUBLIC_*`.
3. Read data on the server after enforcing authorization. Never rely on hidden client buttons or route groups as access control: route groups do not change URL paths or protect routes.
4. Put read-only external handlers in `route.ts`; for writes, use authenticated Server Actions or route handlers. Re-check authentication, authorization, ownership and payload validation **inside each mutation**, not only in page render or middleware.
5. When reading `params`, `searchParams`, `cookies()` and `headers()`, follow Next.js 15 asynchronous APIs; await the corresponding promise. Do not copy synchronous Next.js 14 snippets.
6. Cache per data sensitivity. Don't globally cache personalized tenant data. Explicitly scope reads by the authenticated tenant/user. Handle unauthenticated responses without disclosing whether another user's record exists.
7. Use semantic buttons/labels, keyboard navigation and meaningful loading/empty/error states. Avoid client effects for data fetching that can be done in a Server Component.

## SQLite design and migrations

- Every persisted table has a stable primary key. Use UTC epoch integers or documented ISO timestamps consistently. Include `created_at` / `updated_at` where lifecycle matters.
- Store cents as integers for money, never floating-point dollars. Store user and tenant IDs on rows that need ownership, with indexed foreign keys and uniqueness constraints matching business invariants.
- Parameterize **all user input**. Never build SQL using untrusted string interpolation. For dynamic column names, use an allowlist.
- Enable `PRAGMA foreign_keys = ON` on **each new local connection**. SQLite's default is not a deployment guarantee. Enable WAL on persistent, local files when suitable; do not set it blindly on remote/libSQL adapters.
- Wrap multi-statement writes in transactions; use uniqueness constraints for idempotency and handle contention rather than assuming two requests cannot race.
- **Migrations are append-only.** Add a new numbered migration; never edit, delete or reorder migrations already applied/shared. Forward migrations should be reviewed, tested against an empty DB and a copy of a populated DB, and backed up before production execution.
- Prefer expand–migrate–contract for destructive schema changes: add nullable columns or new tables, backfill, ship compatible readers/writers, then drop old shape only in a separate approved release. SQLite often rebuilds tables for ALTER operations; verify indexes, triggers, FKs and row counts after a rebuild.
- Apply database migrations in an explicit deployment/release step, **not** on app startup, request handling, or `next build`. For Turso, execute via the configured migration tool and respect remote credentials/latency.
- Never commit `.db`, `.sqlite`, `.sqlite3`, their `-wal`/`-shm` files, production dumps or real `.env` files. Check `.gitignore` before creating local data.

## Security and multi-tenancy

- Treat all form, query, webhook and API payloads as untrusted. Validate type, size and shape server-side and return deliberate errors; do not leak stack traces or raw SQL.
- Authorize **each** read and write against the authenticated principal. A guessed record ID must not retrieve another tenant's data. Prefer `WHERE tenant_id = ? AND id = ?` rather than a lookup followed by client-side filtering.
- Use secure session cookies and CSRF protections provided by the chosen auth framework; for custom cookie auth, implement CSRF/origin validation and document the threat model. Don't improvise session crypto.
- Rate-limit login, invitation and public write endpoints with a durable mechanism suited to the deployment. Use server-side verification for webhook signatures and idempotency keys.
- Avoid logging passwords, tokens, API keys, financial account details or personal records. Keep `.env.example` names with dummy placeholders only.
- Do not claim that SQLite supports multi-region distributed writes. A local file requires persistent storage and a single coordinated writer strategy; serverless ephemeral disks are not durable databases.

## Tests and acceptance

- Unit-test validators, permission predicates, and pure domain functions; integration-test actual migrations, constraints and query scopes using a disposable SQLite DB.
- Test unauthorized, wrong-tenant, malformed, missing and duplicate requests. Confirm a two-user fixture cannot cross tenant boundaries.
- For migrations, assert fresh-install success and upgrade from the previous schema with data preserved; run `PRAGMA foreign_key_check` after migration in local SQLite.
- Run lint, strict typecheck, tests and production build before declaring work complete. If dependencies, services or credentials are unavailable, report exactly what was and wasn't executed.
- For UI changes, check mobile layout, keyboard interaction and loading/error/empty states. Describe any inaccessible integration test rather than inventing a green result.

## Change protocol for agents

1. Inspect the impacted files, schema, migration history and package scripts; describe the smallest safe change.
2. Modify only relevant files; write or update tests with the fix. Do not format or rename unrelated code.
3. For DB changes, create a new migration, demonstrate rollback/recovery strategy, and verify on disposable data. Never touch production DBs without explicit operator approval.
4. Run feasible checks and report exact outcomes, known limitations and files changed. Summarize decisions that affect security, data persistence or deployment.

## Anti-patterns (and why)

| Never do this | Reason |
| --- | --- |
| Import DB code into a Client Component | Exposes or breaks server-only code |
| Trust UI hiding or middleware alone for write auth | Any callable Server Action must enforce permissions itself |
| Run arbitrary SQL string concatenation | Enables injection and corrupt data |
| Edit an existing released migration | Makes schema history nondeterministic |
| Store user data in the repository or ephemeral runtime | Leaks data or loses it on redeploy |
| Share the same mutable SQLite file across independent serverless instances | Causes inconsistent state and locking failures |
| Convert monetary values to floats | Introduces rounding errors |
| Introduce a new service or dependency before reading the project | Adds cost, complexity and drift |
| Claim tests passed without execution evidence | Prevents meaningful review |
