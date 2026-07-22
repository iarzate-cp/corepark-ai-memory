---
name: CorePark DDL comment style — reference from Release 171 (EV Charging)
description: The canonical style for DDL scripts at CorePark, extracted from a production-shipped DDL. Governs COMMENT ON TABLE / COLUMN / CONSTRAINT / INDEX + file headers. Israel shared this to align the coupons DDL with the house standard
type: reference
originSessionId: ec870606-5ede-4997-aea2-d3146b28d813
---
**Reference file:** `/Users/israel/Downloads/DDL_RELEASE_171_EV_CHARGING_MODULE.sql`
(EV Charging Module — already shipped to production).

## Golden rules for `COMMENT ON TABLE`

1. **Open with what the table IS, in present tense, one sentence.**
   Examples from Release 171:
   - "Registry of product features that use dynamic configuration."
   - "Global charger catalog maintained once by CorePark."
   - "One complete charge desire — from the moment the vehicle enters the EV queue until it is disconnected or cancelled."

2. **Then describe how it is used, invariants, or lifecycle** — inline diagrams
   are fine when the state machine is non-obvious. Concrete examples too.

3. **Business "why" is allowed** when it clarifies design intent
   (e.g. "This guarantees consistency: every 'Tesla Wall Connector Gen 3' across
   the network has the same power_kw value.").

4. **"Future:" plans are OK** as forward-looking design notes.

5. **NEVER include:**
   - History ("before this table…", "replaces X", "previously we…").
   - Environment references (DEV vs PROD, staging notes).
   - Team internals (names, sprints, rounds, "attempts").
   - Deployment ordering (that belongs to the file header).
   - Java package / query method references (implementation detail that rots).

## `COMMENT ON COLUMN` style

- Short one-liner unless semantics are non-obvious.
- When lifecycle matters (editable-until, overwritten-by, etc.), spell it out
  inline — Release 171 has long column comments where warranted
  (`ticket_ev_data.initial_level_percent` is the archetype).
- Trivial columns get a single sentence:
  - `id`: 'Auto-incremented surrogate key.' or 'DBA-controlled identifier.'
  - `created_at`: 'Record creation timestamp (UTC).'
  - `deactivated_at`: 'Soft delete. NULL = active.'

## `COMMENT ON CONSTRAINT` style

- One sentence explaining WHY the constraint exists — only when the reason
  isn't obvious from the name and definition.
- Examples:
  - "Zero or negative power is physically impossible."
  - "Cannot plug in before entering the queue."
  - "RESTRICT — parking areas with charger assignments must use soft delete
     (deleted_at) instead of physical DELETE."

## `COMMENT ON INDEX` style

- One line: what query pattern it serves.
- Examples:
  - "Find all sessions for a given ticket."
  - "A spot can only have one active charger at a time — prevents
     double-assignment conflicts."

## File header (`/* Purpose */`) style

- Describes the feature being introduced, no history.
- Lists operational layers / architectural components.
- Explicit list of objects created.
- Prerequisites (other DDLs that must run first).
- SDUI registration section when applicable.
- Environment / Safety notes ONLY when the script has env-dependent behavior
  (e.g. `IF EXISTS` guards) — describe the operational fact ("only in DEV, not
  in PROD"), not the history behind it.

## What went wrong in the coupons DDL (first pass) and got fixed

The original coupon `COMMENT ON TABLE` comments framed the new tables as
"replaces the old X column" — a comparative narrative rooted in the intermediate
DEV attempt. That leaked history and DEV-only state into the schema. Rewrite
followed this style guide: opened with what each table IS, described the
lifecycle, kept the business "why" (coupons = credit → compliance), dropped
every historical / environment / team-internal reference from schema-level
comments. Environment note in file 04 was minimalized to the operational
fact only ("legacy columns exist only in DEV — not present in PROD").
