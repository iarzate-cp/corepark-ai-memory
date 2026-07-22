---
name: DDL / SQL conventions — pulled out of CLAUDE.md, pending relocation
description: The Schema-changes (DDL) rules used to live in CLAUDE.md; jvalencia-cp asked to remove them because CLAUDE.md is Java-only. Rules are archived here and pending relocation to Israel's own docs folder
type: reference
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
# DDL / SQL conventions — pending relocation

**Status:** removed from `CLAUDE.md` on 2026-07-17 (commit `51f83e8`). CLAUDE.md is Java-service conventions only.

**Why removed:** jvalencia-cp reviewed the coupons PR (`feature/coupons`) and left the comment: *"Todo estas reglas son para el SQL, no para el servicio tal cual... dejemos aquí solo lo de Java."* Israel agreed and asked to strip them from `CLAUDE.md` now, and re-standardize them later in his own docs.

**How to apply:** if the user starts a schema-change task and asks about conventions, remind them the rules are pending relocation and offer to help move them once they pick the target location.

## Archived rules (verbatim, from removed CLAUDE.md section)

The 102-line section is preserved in the git history at commit `2e24a13` (added) → removed at `51f83e8`. To recover the exact text:

```bash
cd /Users/israel/Dev/Back-End/ms-backoffice-service
git show 2e24a13:CLAUDE.md | awk '/^## Schema changes \(DDL\)$/{f=1} /^## Controllers, responses & errors$/{exit} f'
```

## What the rules covered

- **Location**: `Back-End/ddl/DDL_YYYY_MM_DD_<FEATURE_UPPERCASE>/NN_<ACTION>_<TARGET>.sql`
- **Per-file structure**: Purpose block → Rollback instructions → Script → Grant privileges → Commit
- **Split additive from destructive** files (`NN_` prefix reflects execution order; destructive files land in the same window as the app refactor that stops touching removed columns)
- **Sequences**: always explicit `CREATE SEQUENCE`, never `GENERATED ALWAYS AS IDENTITY` (service peeks `nextval` sometimes)
- **Naming**: `PK_<table>`, `FK_<child>_<parent-purpose>`, `UQ_<table>_<cols>`, `CHK_<table>_<purpose>`, `IX_<table>_<cols>`
- **Comments** on every object — TABLE (multiline), COLUMN, CONSTRAINT, INDEX, SEQUENCE
- **GRANTs** in every file that creates a new table/sequence: order dev WS (`dewsdbcp`) → prod WS → prod analytics
- **`IF NOT EXISTS`** only in rollback ADDs; creative operations fail loudly on collision
- **`COMMIT;` at the end**, no explicit `BEGIN;`

## Target relocation (to be decided by Israel)

Candidates suggested when the rules were pulled:
- `~/Documents/` — for a shareable HTML/MD guide
- `~/Dev/Features/` — mixed with feature write-ups (probably not, that folder is per-feature)
- A dedicated `~/Dev/Conventions/` or similar folder

Await Israel's decision before proposing where to write them.

## Related

- `project_migration_process.md` — how DDL scripts get applied (manual, dewsdbcp lacks DDL rights, use `postgres`)
- `project_coupon_audit_ddl_2026_07_15.md` — the most recent feature that shipped DDL under these conventions
