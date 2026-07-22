# Memory Index — DDL_2026_07_15_COUPONS

- [Release origin, design arc, and post-deploy state](project_release_context.md) — the 4 scripts exist because the "Elias round" quick-fix (three VARCHAR/TIMESTAMPTZ columns on company.coupon, DEV-only, 2026-07-09) was rejected in favor of dedicated audit tables + soft-delete. Applied in DEV on 2026-07-16 alongside the paired app refactor (feature/staging in both repos). PROD deploy pending
- [CorePark DDL comment style guide](reference_ddl_style_guide.md) — reference extracted from the shipped Release 171 EV Charging DDL. Governs headers + COMMENT ON TABLE/COLUMN/CONSTRAINT/INDEX. Rule: describe what the object IS, no history/env/team-internals in schema-level comments
