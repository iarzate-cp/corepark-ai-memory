---
name: Verify schema thoroughly upfront, don't ration verification queries
description: When starting a feature that depends on DB schema, run as many verification queries as needed before writing code. Israel prefers thorough upfront verification over iterative "code → hit surprise → refactor" cycles
type: feedback
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
When starting a feature that depends on DB schema (columns, types, indexes, FKs, defaults, actual data distribution), run all the verification queries the design needs — don't ration them to save round-trips.

**Why:** on 2026-07-08 during coupon feature kickoff, I ran ~6 verification queries against `coreparkdev` (columns, indexes, FKs, partner scoping, voucher statuses, coupon types) before writing a single line of code. Israel's response: *"no te preocupes por todos los queries que necesites que confirmemos. Esto nos va a dar un resultado robusto sin tener que estar iterando"*. He explicitly endorsed the "many upfront queries" approach as producing a more robust design than starting to code with gaps and iterating.

Related but distinct from `feedback_verify_schema_before_migration.md`, which is specifically about proposing DDL. This one is broader: for any new feature whose service/DAO code depends on schema shape, verify the shape first — even if it takes 5+ separate queries.

**How to apply:**
- Before coding a DAO/service that depends on a set of tables, list every schema fact the design assumes (column names, types, nullability, defaults, indexes, FKs, referential integrity, data distributions). If any comes from code inference or a legacy script — verify it against the target env.
- Ship the queries in batches when possible (single query with `IN ('t1', 't2', ...)` for columns), but don't shy away from follow-ups when a screenshot only shows part of a result or a nuance emerges.
- Fold every finding into the relevant `project_*_data_model.md` memory so future sessions read verified truth, not inferred structure.
- Cost calculus: each verification query is minutes. Each "started coding, hit schema surprise, refactor" is a review cycle. Israel has said the former is worth many of the latter.
- When Israel is running the queries manually (no DB MCP), keep the queries copy-pasteable and small — one focused query per screenshot avoids scroll issues.
