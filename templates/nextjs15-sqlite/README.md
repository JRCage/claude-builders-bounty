# Next.js 15 + SQLite SaaS — Claude Code project context

**Bounty:** [#2](https://github.com/claude-builders-bounty/claude-builders-bounty/issues/2). This template is designed for a new Next.js 15 App Router + SQLite SaaS project. It prescribes a Node-backed better-sqlite3 setup or a separately configured hosted Turso/libSQL setup, not both.

## Setup (3 steps)

1. Create a Next.js 15 App Router TypeScript starter with a lockfile and choose a persistent SQLite adapter. Do not run a native SQLite driver on Edge/serverless ephemeral storage.
2. Copy the adjacent `CLAUDE.md` into the **root** of the starter as `CLAUDE.md`.
3. Start Claude Code in that root and ask: **"Identify this project's runtime, folder conventions, DB migration policy, and the checks required before a PR. Do not edit files."** Confirm the answer cites the template's rules and actual package.json, and rejects guessing unavailable scripts or credentials.

## Review checklist

- [ ] Agent correctly defaults to Server Components and understands Next.js 15 async request APIs.
- [ ] Agent selects exactly one SQLite adapter and states a durable storage deployment assumption.
- [ ] Agent forbids editing applied migrations, and requires backups/rehearsals for production.
- [ ] Agent enforces tenant authorization and parameterized SQL on all mutations.
- [ ] Agent does not invent scripts, migrations, package dependencies or test results.

**Validation note:** This contribution was structurally checked for its required sections, but no authenticated Claude Code test session or production SQLite deployment is claimed. The checklist above is an acceptance procedure for reviewers, not a fabricated transcript.

**AI-assisted contribution:** Written and reviewed with AI assistance for the connected GitHub account. No paid API usage is required to use the template.
