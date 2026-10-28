# Next.js 15 + SQLite CLAUDE.md template (bounty #2)

Copy [CLAUDE.md](CLAUDE.md) to the root of a **new** Next.js 15 App Router + SQLite SaaS repository. The template establishes an explicit backend, tenant-security, database-migration and frontend architecture with reasons for each decision.

## Setup (3 steps)

1. Scaffold a new TypeScript Next.js 15 App Router project with `npx create-next-app@15 my-saas --ts --app --eslint --src-dir --import-alias "@/*"`.
2. Copy `templates/nextjs-sqlite/CLAUDE.md` to `my-saas/CLAUDE.md`; configure `better-sqlite3`, Vitest and `tsx` only when implementing features requiring them.
3. Open the project in Claude Code and ask it to add a tenant-scoped invoice feature. It should follow the template without asking which database, runtime, folder layout or authorization design to use.

## Verification notes

This submission is an instruction template, not a bootstrapped running SaaS. Static validation checks that the supplied template contains all requested sections and explicit decisions. **The interactive Claude Code smoke test has not been run in this environment**, and the template does not claim it has. The full acceptance criterion of confirming behavior in Claude Code remains for reviewer verification.

## Expected Claude Code smoke test

Prompt: `Create a tenant-scoped invoices CRUD module using the project's CLAUDE.md. Include the migrations, validation, auth checks, integration tests, and state which commands you ran.`

Verify: files appear in `src/server/db/migrations`, `src/server/repositories`, `src/server/services`, `src/app`; SQL is parameterized; other-tenant reads are denied; migrations are explicit; no SQLite import leaks to an edge/client component; failed commands are reported, not fabricated.

AI-assisted preparation disclosed. No live secrets, real payments, or privileged commands.
