---
name: Corepark backend DDL convention — Back-End/ddl/ folder, one folder per feature, no Flyway/Liquibase
description: Schema changes are manual, but they DO have a convention now — versioned SQL files in Back-End/ddl/DDL_YYYY_MM_DD_<FEATURE>/, atomic files with rollback blocks, GRANTs to prod+dev users. Reference script sets the style.
type: reference
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
**Fact:** The Corepark backend services (including `ms-backoffice-service`) do NOT use Flyway, Liquibase, or any migration framework. The user (Israel) runs schema changes manually via a DB client (JetBrains DataGrip / IntelliJ / psql, etc.). But **as of 2026-07-15, there IS a convention for where and how the SQL lives** — see below.

## Folder structure

DDLs live at `Back-End/ddl/` (repo-root sibling of the microservices). One subfolder per feature:

```
Back-End/ddl/
  DDL_YYYY_MM_DD_<FEATURE_UPPERCASE>/
    01_<ACTION>_<TARGET>.sql
    02_<ACTION>_<TARGET>.sql
    03_<ACTION>_<TARGET>.sql
```

- Folder name: `DDL_` prefix + ISO date with underscores + uppercase feature slug. E.g. `DDL_2026_07_15_COUPONS`.
- Files: `NN_` prefix (order of execution) + uppercase action verb + target. E.g. `01_CREATE_COUPON_REQUEST.sql`, `03_ADD_COUPON_DELETED_AT.sql`, `04_DROP_LEGACY_COUPON_COLUMNS.sql`.
- Split by concern: additive files are safe-anytime; destructive files (DROP COLUMN / DROP TABLE) come last and depend on the code refactor being deployed first.

## Per-file structure (matches `/Users/israel/Downloads/DDL_RELEASE_171_EV_CHARGING_MODULE.sql` — the canonical reference)

Every SQL file has these sections in order:

1. **Purpose block** at the top — C-style `/* ... */` block with `Purpose:`, `Objects created/modified:`, `Safety:` (ADDITIVE vs DESTRUCTIVE), `Ordering:` (dependencies), `Release:` (date + feature name).
2. **`/*******************ROLLBACK INSTRUCTIONS***********************/`** — every DROP/ALTER statement needed to reverse the file, commented out with `--` so the reviewer can copy-paste. For destructive files, add a `WARNING — data loss is irreversible` note.
3. **`/**********************SCRIPT**********************************/`** — the actual DDL.
4. **`/******************GRANT PRIVILEGES****************************/`** — GRANTs for dev and prod users (only for files that create objects; ALTER-only files skip this).
5. **`/**********************COMMIT**********************************/`** — literal `COMMIT;` at the end wrapping everything (no explicit `BEGIN;` — psql runs DDL in an implicit transaction).

## Naming conventions inside DDL

- `CREATE SEQUENCE company.<table>_id_seq AS BIGINT START WITH 1 INCREMENT BY 1 MINVALUE 1 MAXVALUE 9223372036854775807 CACHE 1;` — explicit sequence, never `GENERATED ALWAYS AS IDENTITY`. The service layer sometimes peeks nextval before insert.
- `id BIGINT NOT NULL DEFAULT nextval('company.<table>_id_seq')` in the column.
- `ALTER SEQUENCE company.<table>_id_seq OWNED BY company.<table>.id;` at the end so DROP TABLE also drops the sequence.
- Constraints: `PK_<table>`, `FK_<child>_<parent-purpose>`, `UQ_<table>_<cols>`, `CHK_<table>_<purpose>`. Uppercase prefix.
- Indexes: `IX_<table>_<cols>` (or `UQ_<table>_<cols>` for unique indexes). Uppercase prefix.
- Composite FKs: add `MATCH FULL` (required when FK columns can be nullable together).
- FK behaviors: `ON DELETE CASCADE ON UPDATE CASCADE` for owned relationships; `ON DELETE RESTRICT ON UPDATE CASCADE` for catalog-ish references.
- `NOW()` (uppercase) in defaults, not `now()`.
- `TIMESTAMPTZ` (not `timestamp with time zone`).

## Comments

Every schema object gets an inline `COMMENT ON`:

- `COMMENT ON SEQUENCE` — one-liner is fine.
- `COMMENT ON TABLE` — **multiline** using literal string with `\n` (leading newline + paragraphs separated by blank lines). Explain what problem the table solves, how the app interacts with it, why it exists (originating use case).
- `COMMENT ON COLUMN` — for **every** column without exception (`id`, scope columns, all business columns, timestamps).
- `COMMENT ON CONSTRAINT` — for every FK, CHECK, and UQ. Explain the enforcement guarantee and edge cases (why RESTRICT vs CASCADE, what the CHECK protects against).
- `COMMENT ON INDEX` — for every index. Name the access pattern it serves.

## GRANTs (bottom of each file that creates objects)

Three grants total, always in the order: dev WS → prod WS → prod analytics. Dev has NO read-only user — the WS user is the only role that gets grants there.

```sql
-- Development web service user (no read-only user in development)
GRANT SELECT, INSERT, UPDATE, DELETE ON company.<table> TO dewsdbcp;
GRANT USAGE, SELECT ON SEQUENCE company.<table>_id_seq TO dewsdbcp;

-- Production web service user
GRANT SELECT, INSERT, UPDATE, DELETE ON company.<table> TO prwsdbcp;
GRANT USAGE, SELECT ON SEQUENCE company.<table>_id_seq TO prwsdbcp;

-- Production analytics user (read-only)
GRANT SELECT ON company.<table> TO prbadbcp;
```

**All three usernames confirmed by the backend team on 2026-07-15:**
- `dewsdbcp` — dev web service (RW), the only role in dev.
- `prwsdbcp` — prod web service (RW).
- `prbadbcp` — prod analytics (read-only, always).

No intermediate environments (QA / staging) — those use the same users as dev or prod depending on the deploy target. Granting to a role that does not exist aborts the whole transaction (`ERROR: role "<name>" does not exist`); update this list if the DBA team ever introduces a new role.

## How to apply

- **DDL files are NOT committed to a git repo.** Israel drafts them locally under `Back-End/ddl/DDL_YYYY_MM_DD_<FEATURE>/`, then shares the files directly with the backend team, who authorizes and applies them against each environment. The local folder is a staging area, not source of truth.
- Each file is applied manually per environment (local → dev → prod). No automated migration runner.
- Files are meant to be reviewed one by one — split additive vs destructive so the reviewer can approve incrementally.
- Schema drift between environments is real (a column added to dev in an earlier release may never have been propagated to prod). When something breaks in dev, "did the backend team apply this file on dev too?" is a valid first question.
- Never inject `BEGIN` at the top of files — the `COMMIT;` at the end works with psql's implicit transaction. Adding `BEGIN;` alongside the header creates a nested-transaction warning.

### Survive env drift with a single script

- **Destructive operations use `IF EXISTS`.** `DROP TABLE IF EXISTS`, `DROP COLUMN IF EXISTS`, `DROP SEQUENCE IF EXISTS`, `DROP INDEX IF EXISTS`. In environments where the object was never introduced, the statement becomes a silent no-op instead of aborting the transaction with `ERROR: <object> does not exist`.
- **Rollback ADDs use `IF NOT EXISTS`.** For re-adding columns as a rollback, `ADD COLUMN IF NOT EXISTS <name> <type>` covers environments that still have the column.
- **Creative operations do NOT use `IF NOT EXISTS`.** `CREATE TABLE`, `CREATE SEQUENCE`, `CREATE INDEX`, `ADD COLUMN` should fail loudly on unexpected collision — silently no-op'ing a pre-existing object hides real bugs (someone else already created it, or the DDL was applied twice).

Real example this rule was written from: the coupons feature dropped `requested_by`, `downloaded_at`, `downloaded_by` from `company.coupon`. Those columns only existed in DEV (added in the earlier Elias round, never deployed to PROD). Without `IF EXISTS`, the destructive file would have aborted in PROD.

## Related

- Canonical reference: `/Users/israel/Downloads/DDL_RELEASE_171_EV_CHARGING_MODULE.sql` (EV Charging Module release).
- `reference_backoffice_backend_claudemd.md` for other backend conventions (Lombok, `ApiResponse<T>`, JVM 21).
