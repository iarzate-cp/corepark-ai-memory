---
name: DDL_2026_07_15_COUPONS release — origin and design arc
description: The release exists because the earlier "Elias round" quick-fix (three VARCHAR/TIMESTAMPTZ columns on company.coupon in DEV) was rejected by a final analysis in favor of dedicated audit-history tables + soft-delete. Explains why the 4 scripts look the way they do
type: project
originSessionId: ec870606-5ede-4997-aea2-d3146b28d813
---
Historical arc that produced the 4 scripts in this directory:

**1. 2026-07-08 pm — full coupon CRUD shipped on `feature/coupons`** in both
`ms-backoffice-service` and `frontend-commerce`, unpushed. Delete was implemented
as **hard-delete with USED-lock** because the DBA was unavailable for DDL and
Israel asked for "no DB changes". Trade-off explicitly documented: revisit with
soft-delete when the DBA returns.

**2. 2026-07-09 — "Elias round" quick-fix landed in DEV only:**

```sql
ALTER TABLE company.coupon
    ADD COLUMN requested_by  VARCHAR(100),
    ADD COLUMN downloaded_at TIMESTAMP WITH TIME ZONE,
    ADD COLUMN downloaded_by VARCHAR(100);
```

Applied to unblock an earlier round of the feature. Never propagated to PROD.

**3. Final analysis rejected the Elias approach.** Reasons:
- Three flat columns on `company.coupon` only remember the LAST event and overwrite everything before.
- `VARCHAR` for who-did-what has no referential integrity — cannot tie to a real `company.employee` row.
- Only CREATE and download were traced; APPEND, EDIT and DELETE had no persisted trace at all.
- Coupons behave like credit at valet parkings, so this is a compliance concern, not ergonomics.

**4. 2026-07-15 (this release)** — the redesign the final analysis produced:
- `company.coupon_request` — full history of CREATE/APPEND/EDIT/DELETE events, immutable rows, composite FK to `company.employee` (`operator_company_id + employee_id`).
- `company.coupon_download` — one row per XLSX export, same FK pattern.
- `company.coupon.deleted_at` — enables soft-delete, so audit history outlives the batch. Replaces the earlier hard-delete-with-USED-lock design.
- File 04 drops the three Elias columns via `DROP COLUMN IF EXISTS` — destructive in DEV, silent no-op in PROD.

**5. 2026-07-16 — release applied in DEV.** Files 01/02/03/04 ran clean as `postgres` from AWS Dev Intranet. Paired application refactor shipped in the same window:
- `ms-backoffice-service`: commit `2e24a13` on `feature/coupons`, merged to `feature/staging` (`1587c86`). 58/58 tests green.
- `frontend-commerce`: commit `be97cf4` on `feature/coupons`, merged to `feature/staging` (`15a14d8`). Build clean.
- Power-user testing on `feature/staging` (DEV) confirmed all flows: create, add more, edit, delete (soft), download, and history.

**6. PROD deploy — pending.** When it happens the DBA gets `DDL_RELEASE_COUPONS_AUDIT_HISTORY.sql` (consolidated PROD script; no `dewsdbcp` grants, no `DROP COLUMN` section since PROD never had the Elias columns). `DEV_CLEANUP_LEGACY_COUPON_COLUMNS.sql` stays DEV-only and is not run in PROD.

**How to apply this to future work in this directory:**
- Comments MUST NOT frame the new tables as "replacing" the Elias columns without acknowledging that those columns only existed in DEV. Read as-if-in-PROD, "replaces" is misleading. The correct framing is "introduces audit history that did not exist before; in DEV, also supersedes the short-lived Elias experiment".
- The 5 problematic framings identified during review (headers + `COMMENT ON TABLE` of files 01 and 02, plus the opening of file 04) were rewritten on 2026-07-16 with the correct framing.
- The `Environment note` in file 04 is the source of truth for DEV-vs-PROD divergence — reference it from other files instead of duplicating.
- Ordering constraint (already honored in the DEV run): in DEV, file 04 must run AFTER files 01, 02, 03 AND after the paired application refactor. In PROD, ordering vs. the app refactor is not sensitive (nothing to drop, nothing that reads the Elias columns).
- The `CHK_coupon_request_action` CHECK stays at 4 values (`CREATE, APPEND, EDIT, DELETE`). A brief attempt on 2026-07-16 to add `ACTIVATE` (for a Deactivate/Activate rename with reactivation) was reverted — Israel chose not to touch the DB again after the release.

## Related deliverables

- Power-user operator guide: `/Users/israel/Documents/coupons-operator-guide-2026-07-16.html` — 11 sections walking through every flow (listing, employee attribution, create, add more, edit, delete, download, history, notifications, error codes, reporting). Matches the style of the earlier Survey Admin Operator Guide.
- Backend project memory: `~/.claude/projects/-Users-israel-Dev-Back-End-ms-backoffice-service/memory/project_coupon_audit_ddl_2026_07_15.md` — shipped state of the app refactor, endpoint catalog, design decisions.
- Frontend project memory: `~/.claude/projects/-Users-israel-Dev-frontend-commerce/memory/project_coupons_module.md` — shipped state of the FE integration.
