---
name: DB migration process — manual scripts, no Flyway/Liquibase
description: Backend team runs SQL scripts by hand; new scripts must be idempotent, self-contained, and require DDL privileges the app user does not have
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
Backend team confirmed on 2026-07-01: **there is no migration framework (Flyway/Liquibase) in `ms-backoffice-service`**. Every schema change is applied by hand against the target DB (dev/staging/prod).

**Why:** legacy Corepark convention — DBAs prefer explicit control over schema changes; no CI-driven migration pipeline exists.

**How to apply:**

- Any DDL change goes in `scripts/sql/<YYYY-MM-DD>-<slug>.sql` (dated filename) in the repo, checked in with the feature branch that needs it. Backend team runs it manually before/after merge.
- Scripts MUST be:
  - **Idempotent** — wrap DDL where possible (`ADD COLUMN IF NOT EXISTS`, `NOT EXISTS` guards on backfills)
  - **Transactional** — `BEGIN ... COMMIT` so partial failure rolls back
  - **Self-documenting** — top comment block with Purpose / Safety / Impact so DBAs can review without asking
- The app runtime user (`dewsdbcp`) does **NOT** have DDL privileges — tables are owned by `postgres`. Local dev applies scripts as `postgres` via IntelliJ Database tool or psql; shared envs are applied by backend team.
- Do **not** assume a script has been applied just because it's in the repo — confirm via `information_schema.columns` / `pg_tables` before writing code that depends on new columns.

**How to apply this rule to future work:**
- When designing a feature that needs schema changes, plan the SQL script alongside the code
- Frontend code that depends on a new column/table should degrade gracefully (guard on missing fields) until the migration lands in the target env
- Coordinate with backend team early to schedule the manual run
