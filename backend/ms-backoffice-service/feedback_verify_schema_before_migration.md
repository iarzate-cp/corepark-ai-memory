---
name: Verify DB schema before proposing migrations
description: Before drafting an SQL migration, query pg_indexes / information_schema on a target env. Don't infer from code alone
type: feedback
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
Before proposing a DB migration (index, column, constraint), query the actual schema — do not infer "this must not exist" from reading application code.

**Why:** on 2026-07-02, I proposed a unique partial index on `company.survey (op, loc) WHERE deactivated_at IS NULL` after spotting a TOCTOU race in `SurveyAdminServiceImpl`. The DBA reviewing the migration pushed back: the index already existed in all environments (dev/staging/prod). My method was code-review only; I never ran a `\di` or `SELECT * FROM pg_indexes ...`. The script was redundant — a no-op thanks to `IF NOT EXISTS` — but wasted the DBA's review cycle.

**How to apply:**
- For any proposed DDL, first run a schema-check query on at least one target env. Examples:
  - Indexes: `SELECT indexname, indexdef FROM pg_indexes WHERE schemaname = '<schema>' AND tablename = '<table>';`
  - Columns: `SELECT column_name, data_type FROM information_schema.columns WHERE table_schema = '<schema>' AND table_name = '<table>';`
  - Constraints: `SELECT conname, pg_get_constraintdef(oid) FROM pg_constraint WHERE conrelid = '<schema>.<table>'::regclass;`
- If Israel doesn't have DB access handy, ask him to run it (or delegate to a DBA) BEFORE drafting the SQL file. Otherwise say explicitly: "I'm proposing this without schema verification — please confirm the index/column/constraint isn't already present before applying."
- Even after verification, keep migrations idempotent (`IF NOT EXISTS`, `IF EXISTS`) — but idempotency is a safety net, not a substitute for checking.
- Application-level defenses (handlers, exception mappings) can still ship without the migration when the constraint already exists in DB — they add value by translating DB errors to friendly response codes.
