---
name: UUID Redirect Flow — Origin & Backend Migration
description: Why `uuidSetter()` in `core/utils/uuid.ts` exists (three families of operational problems it solves) and why it is being moved out of the frontend into ms-valet-service, with a Backoffice admin CRUD in ms-backoffice-service.
type: project
---

Documents the **origin** of the guest-page UUID redirection flow (the `uuidSetter()` chain in `src/app/core/utils/uuid.ts`) and the rationale behind moving it to the backend. Written 2026-07-29 while planning the migration.

## What the current flow does

At page load, before the first HTTP call, the frontend rewrites the pair `(rawLocationUUID, ticket)` — parsed from the URL — into a **canonical** `locationUUID` using a hardcoded chain of `if`s in `core/utils/uuid.ts` (`uuidSetter`). Every downstream call, URL rewrite (`router.navigate([/ticket/${uuid}/${ticket}])`), and comparison (`urlParameters.uuid.includes(...)`) uses the remapped value. Only one endpoint on the wire actually receives the UUID: `GET /partner/guest/ticket-info/{uuid}/{ticket}` — everything else resolves by `operatorCompanyId` / `parkingLocationId` returned in that first response.

## Why this exists — three families of operational problems

The `uuidSetter` chain was not born as a design decision; it accreted over years to solve real operational messes that cannot be fixed at the source:

### 1. Multi-tenant printed QRs (majority of the rules)

A single physical QR — printed on tickets, signs, or valet stands — is used to serve **several neighboring locations** by dividing the ticket-number range. The QR encodes one `location_uuid`; the ticket number picks which real location the guest belongs to. Cannot be undone because the physical material is already in the field.

Examples from `uuidSetter`:
- `Porzana` (`56a24c83-…`) → `SpoonStable` for tickets `11500–11750`; → itself for others.
- `ButcherAndTheBoar` → `Porzana` for tickets `10500–11000`.
- `MarinValet` → `HamptonInnRiverside` for `13000–13999`, → `DowntownDowney` for `15000–15999`.
- `Josefina` → `SpoonStable` for `11000–11250`.
- `PrivateEvents` → `InspirationAwards` for `12000–12750`.
- `ArmatureWorks` → `MarketStreet` for `1000–1999`.
- `SanJuanRiverStreetMarketplace` → different Howelite lots by range.
- `QRCodeTestLocation` (Safety Park) fans out to **8 different restaurants** by range: Fia, Spago BH, Scopa, Ivy at the Shore, Steak 48, South Beverly Grill, etc.
- `SennaHouse` ↔ `WScottsdale` cross-mapping by range.

### 2. Location UUID migration under a live printed QR

An existing QR needs to be repointed to a **different `location_uuid`** without reprinting. Currently a single case:
- Curbstand: `c7909083-1a7d-4071-9514-d6187a0d075d` → `d8e5a93b-…` (Beaudry).

### 3. Typographical errors on printed material

The UUID was printed **wrong** and reaches the server as an invalid string. Currently one known case:
- `7f64fe6-61a5-4e8d-9638-32ed1a73e6d0` (missing leading `0`) → `07f64fe6-61a5-4e8d-9638-32ed1a73e6d0`.

This is a superset of `project_uuid_redirects.md`, which documents only the zero-padding case; that memory remains valid as a specific instance of family (3).

## What it solves in practical terms

- **Recovers a resolvable `(operatorCompanyId, parkingLocationId)`** from what the guest actually scanned, so `ticket-info` succeeds and the whole guest page renders.
- **Preserves URL identity** after resolution: the canonical UUID is pushed to the address bar via `router.navigate([/ticket/${uuid}/${ticket}])`, so refresh, deep-link, browser back, and localStorage restoration all keep working with a stable key.
- **Keeps operational surprises invisible to the guest**: a guest at Josefina scanning ticket `11100` reaches SpoonStable's config, staff, payment gateway, terms, etc. — with no failure state.

## Cost of keeping it on the frontend

- **Every operational tweak = frontend deploy.** Adding a new range or fixing a misprint requires a code change to `core/utils/uuid.ts`, PR review, and a release to production.
- **Domain leakage into UI code.** The frontend carries ~150 lines of hardcoded location UUIDs and ticket ranges for restaurants and hotels that have nothing to do with rendering.
- **Inconsistent coverage.** Only `GET /partner/guest/ticket-info` benefits from the remap. The newer `GET/PUT /valet/vehicle-info/{uuid}/{ticket}` (in `ms-valet-service` slice `com.corepark.valet.vehicleinfo`) receives the raw URL UUID directly and would fail for any redirected location — safe today only because those locations don't reach vehicle-info yet, but structurally fragile.
- **Dead code paths.** At least one rule in `uuidSetter` is unreachable due to earlier `return`s in the chain (the `MarinValet → Winery` branch after the general MarinValet block returns).
- **String-match on canonical UUIDs.** Two downstream features (`rate-info` for Omni LA overnight copy, `ticket-actions` for Wave Hotel Orlando tip prompt) do `urlParameters.uuid.includes(...)` on the remapped value. If the remap ever misfires, these feature flags silently misfire too.

## Why we are moving it to the backend

- **The source of truth for `location_uuid` is Postgres**, not TypeScript. Backend already resolves UUIDs to IDs on two paths (`GuestPageDao.getInitialCompanyInfo` via `QUERY_GET_OPERATOR_INFO` in `ms-valet-service`, and `GuestVehicleInfoDao.resolveLocationByUuid`). A resolver is the natural home for the remap.
- **A schema-backed table replaces frontend deploys**: adding a new redirect becomes an `INSERT`. Operations/support can eventually maintain their own rows through a Backoffice CRUD.
- **Both entry points converge on the same primitive** — "given the QR UUID + a ticket, return the canonical `(operator_company_id, parking_location_id)`" — so a shared resolver deduplicates the two hand-rolled queries and makes vehicle-info consistent with ticket-info.
- **The `TicketInfo` response already includes `location.uuid`** (`common.ts:15-21`). Frontend can trust that field for URL rewrite / `urlParametersState` / the two `.includes()` branches, and stop computing the canonical UUID itself.
- **The FE diff to close this out is small**: delete `core/utils/uuid.ts`, delete its import from `ticket.component.ts`, and use `ticketInfo.location.uuid` where `this.uuid()` was used. Everything else (state, guards, dialogs, URL rewrite) continues working because they only consume the `{uuid, ticket}` pair, not any decision derived from it.

## Where the migration lands (cross-repo pointers)

- **`Back-End/ms-valet-service`** — the resolver lives here. Two consumers already exist in this repo:
  - `com.corepark.partner.guest.service.impl.GuestPageService#getTicketInfo` — right before `guestPageDao.getInitialCompanyInfo(rawUuid)`.
  - `com.corepark.valet.vehicleinfo.service.impl.GuestVehicleInfoService#{getVehicleInfo,updateVehicleInfo}` — right before `vehicleInfoDao.resolveLocationByUuid(rawUuid)`.
  - Suggested new slice: `com.corepark.valet.locationredirect` (or `partner.commons` if shared with more partner endpoints). Exposes a single method `resolve(rawUuid, ticket) → UUID`.
- **`Back-End/ms-backoffice-service`** — admin CRUD lives here as a new slice `com.corepark.backoffice.locationredirect` following the modern pattern (typed `ApiResponse<T>` + `ErrorCode`, `@RequiredArgsConstructor`, `NamedParameterJdbcTemplate` with `BeanPropertyRowMapper`). No existing slice covers this domain; the nearest structural neighbors are `backoffice.location` (parking-location CRUD) and `backoffice.guestpage` (guest-page config CRUD) — both use the legacy `Response` + `MessageCode` envelope and should NOT be extended for this feature.
- **`Back-End/ddl/`** — new table `company.location_redirect` (full DDL below). The zero-padding case (family 3) is solved by a `sanitizeUuid()` step in the resolver, not as a table row — the input is not a valid UUID and would require a `TEXT` column that contaminates the schema.

## Proposed DDL

Suggested filename: `DDL_RELEASE_XXX_LOCATION_REDIRECT.sql` (release number assigned at merge time). Mirrors the structure and conventions of `DDL_RELEASE_171_EV_CHARGING_MODULE.sql`: header block, `\set` psql variables, rollback comments, one section per object with `COMMENT ON` blocks, GRANT statements at the end, single `COMMIT`.

**Design choices baked in:**
- Lives in `company.` (not `custom.`) because rows target a specific `company.parking_location`. `custom.` is reserved for globally-shared reference catalogs; this table has FK-like semantics into a per-tenant table.
- `source_location_uuid` is intentionally NOT foreign-keyed to `company.parking_location.location_uuid` — several sources are printed UUIDs that do not exist as real locations (e.g. Curbstand's `c7909083-…`). Only the target is FK-enforced.
- `target_location_uuid` FK to `company.parking_location.location_uuid` assumes that column has a UNIQUE constraint. If it does not, the FK falls back to `(target_operator_company_id, target_parking_location_id)` composite — same as the EV script's FK to `parking_location`. This must be verified against the current schema before merge.
- `ticket_range_start` and `ticket_range_end` are nullable INTEGERs. Both NULL = the rule applies to every ticket at this source (covers the Curbstand simple remap). Both set = closed range `[start, end]`. Ranges use INTEGER even though `parking_service.ticket` is VARCHAR, because every real-world source uses numeric tickets; non-numeric tickets fall through the resolver unchanged.
- `priority` (INTEGER, lower = higher priority) resolves conflicts when two rules share a source. Today the JS `if/else if` chain resolves conflicts by declaration order; the table equivalent is an explicit priority column so ordering is data, not code.
- `deactivated_at` for soft delete — matches every other `company.*` table in the reference DDL. Historical rules stay auditable.
- Partial UNIQUE index `(source_location_uuid, ticket_range_start, ticket_range_end) WHERE deactivated_at IS NULL` prevents two active rules with identical shape. It does NOT prevent overlapping ranges with different targets — that would need a GiST + `int4range` EXCLUDE constraint, deferred to a later phase.
- Initial rules from `uuidSetter` are backfilled in a separate script (`DDL_YYYY_MM_DD_LOCATION_REDIRECT_BACKFILL.sql`), matching the split seen in `DDL_2026_07_22_COUPON_REQUEST_BACKFILL`.

```sql
/*
 Purpose: Location Redirect Table — Move guest-page UUID remap out of the frontend.

          Replaces the hardcoded `uuidSetter()` chain in
          frontend-guest-page/src/app/core/utils/uuid.ts with a schema-backed
          resolver in ms-valet-service. Rules are administered via a new
          ms-backoffice-service slice.

          A rule redirects a scanned `source_location_uuid` (printed on QRs or
          tickets) to a `target_location_uuid` (the real location that should
          serve the guest). Optionally scoped to a ticket-number range. The
          resolver is called by:
            - com.corepark.partner.guest — before ticket-info lookup
            - com.corepark.valet.vehicleinfo — before location lookup

          Zero-padded misprint UUIDs (7f64fe6- → 07f64fe6-) are normalized by
          the resolver via sanitizeUuid() BEFORE this table is consulted, so the
          source column can stay strictly typed as UUID.

          Objects created:
            company.location_redirect         — Redirect rules (source → target, optional range)

          Prerequisites (verify before deploy):
            company.parking_location.location_uuid must have a UNIQUE constraint
            (or a unique index) for the FK to work. If not, this DDL falls back
            to the composite FK against (operator_company_id, parking_location_id).

 Release: XXX (Location Redirect Module)
*/

\set vDatabase corepark
\set vWsUser prwsdbcp
\c :vDatabase;

/*******************ROLLBACK INSTRUCTIONS***********************/
/*
 Rollback steps (reverse dependency order):
 1. Drop location_redirect table and sequence
*/

-- 1. location_redirect
-- DROP TABLE IF EXISTS company.location_redirect CASCADE;
-- DROP SEQUENCE IF EXISTS company.location_redirect_id_seq;

/*********************END ROLLBACK*****************************/

/**********************SCRIPT**********************************/

/****** 1. company.location_redirect — Redirect rules ******/

CREATE SEQUENCE company.location_redirect_id_seq
    AS BIGINT
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9223372036854775807
    CACHE 1;

CREATE TABLE company.location_redirect (
    id BIGINT NOT NULL DEFAULT nextval('company.location_redirect_id_seq'),
    uuid UUID NOT NULL DEFAULT uuid_generate_v4(),
    source_location_uuid UUID NOT NULL,
    target_location_uuid UUID NOT NULL,
    ticket_range_start INTEGER,
    ticket_range_end INTEGER,
    priority INTEGER NOT NULL DEFAULT 100,
    note TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ,
    deactivated_at TIMESTAMPTZ,
    CONSTRAINT PK_location_redirect PRIMARY KEY (id),
    CONSTRAINT FK_location_redirect_target
        FOREIGN KEY (target_location_uuid)
        REFERENCES company.parking_location (location_uuid)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT CHK_location_redirect_range_bounds CHECK (
        (ticket_range_start IS NULL AND ticket_range_end IS NULL)
        OR (ticket_range_start IS NOT NULL AND ticket_range_end IS NOT NULL
            AND ticket_range_start >= 0 AND ticket_range_end >= ticket_range_start)
    ),
    CONSTRAINT CHK_location_redirect_source_not_target CHECK (source_location_uuid <> target_location_uuid),
    CONSTRAINT CHK_location_redirect_priority_non_negative CHECK (priority >= 0)
);

ALTER SEQUENCE company.location_redirect_id_seq OWNED BY company.location_redirect.id;

CREATE UNIQUE INDEX UQ_location_redirect_uuid ON company.location_redirect (uuid);
CREATE INDEX IX_location_redirect_source ON company.location_redirect (source_location_uuid) WHERE deactivated_at IS NULL;
CREATE INDEX IX_location_redirect_target ON company.location_redirect (target_location_uuid);
CREATE UNIQUE INDEX UQ_location_redirect_shape_active
    ON company.location_redirect (source_location_uuid, COALESCE(ticket_range_start, -1), COALESCE(ticket_range_end, -1))
    WHERE deactivated_at IS NULL;

COMMENT ON SEQUENCE company.location_redirect_id_seq IS
    'Sequence for surrogate keys of company.location_redirect.';

COMMENT ON TABLE company.location_redirect IS '
Rules that redirect a scanned guest-page UUID to the real location that should
serve the ticket. Replaces the hardcoded uuidSetter() chain that used to live
in the guest-page frontend.

Consumers (ms-valet-service):
  - com.corepark.partner.guest.service.impl.GuestPageService#getTicketInfo
    Calls resolve(rawUuid, ticket) before guestPageDao.getInitialCompanyInfo.
  - com.corepark.valet.vehicleinfo.service.impl.GuestVehicleInfoService
    Calls resolve(rawUuid, ticket) before vehicleInfoDao.resolveLocationByUuid.

Resolution algorithm (service layer):
  1. Normalize the raw UUID (zero-pad missing leading zeros — sanitizeUuid()).
  2. SELECT active rules WHERE source_location_uuid = :normalizedUuid.
  3. Filter by ticket range: rules with NULL range match any ticket; ranged
     rules match when :ticketNumber BETWEEN ticket_range_start AND ticket_range_end.
  4. Order by priority ASC, id ASC. Return the first target_location_uuid.
  5. If no rule matches, return the normalized UUID unchanged.

Rules are administered via ms-backoffice-service (com.corepark.backoffice.locationredirect).
Support/ops teams add or deactivate rules without a frontend deploy.

Three families of rules the table must cover (see project_uuid_redirect_flow.md
for the origin story):
  1. Multi-tenant printed QRs (majority): one source, multiple targets per ticket
     range. Example: Josefina UUID with tickets 11000-11250 → SpoonStable.
  2. UUID migration under a live printed QR: source has no range, always maps
     to target. Example: Curbstand old UUID → Beaudry.
  3. Zero-padding misprints: NOT stored here. Handled by sanitizeUuid() in the
     resolver so the source column stays strictly typed as UUID.

Historical rows (deactivated_at IS NOT NULL) stay in the table for audit —
support can see when a rule was retired and why.
';

COMMENT ON COLUMN company.location_redirect.id IS 'Auto-incremented surrogate key.';
COMMENT ON COLUMN company.location_redirect.uuid IS 'External API identifier for admin CRUD. Use id for internal joins.';
COMMENT ON COLUMN company.location_redirect.source_location_uuid IS 'The UUID as it arrives from the QR / printed ticket. NOT foreign-keyed to parking_location because some sources (e.g. Curbstand''s c7909083-...) are not real location UUIDs — that is precisely why the redirect exists.';
COMMENT ON COLUMN company.location_redirect.target_location_uuid IS 'The canonical location UUID the guest should be routed to. FK-enforced against parking_location so a rule cannot target a location that does not exist.';
COMMENT ON COLUMN company.location_redirect.ticket_range_start IS 'Lower bound of the ticket-number range this rule applies to (inclusive). NULL together with ticket_range_end = rule applies to every ticket at this source.';
COMMENT ON COLUMN company.location_redirect.ticket_range_end IS 'Upper bound of the ticket-number range (inclusive). Must be NULL if start is NULL, otherwise >= start.';
COMMENT ON COLUMN company.location_redirect.priority IS 'Lower value wins when multiple rules match the same (source, ticket) pair. Default 100 leaves room for specific overrides at lower values.';
COMMENT ON COLUMN company.location_redirect.note IS 'Free-form note for ops/support (e.g. "Josefina overflow into SpoonStable during Dec 2025 events"). Not shown to guests.';
COMMENT ON COLUMN company.location_redirect.created_at IS 'Record creation timestamp (UTC).';
COMMENT ON COLUMN company.location_redirect.updated_at IS 'Last update timestamp (app-managed).';
COMMENT ON COLUMN company.location_redirect.deactivated_at IS 'Soft delete. NULL = active rule considered by the resolver.';

COMMENT ON CONSTRAINT FK_location_redirect_target ON company.location_redirect IS 'RESTRICT — a location referenced by an active redirect rule cannot be hard-deleted; the rule must be retired first.';
COMMENT ON CONSTRAINT CHK_location_redirect_range_bounds ON company.location_redirect IS 'Ranges are all-or-nothing (both NULL or both set) and non-empty (end >= start >= 0). Prevents half-defined ranges that would silently misbehave in the resolver.';
COMMENT ON CONSTRAINT CHK_location_redirect_source_not_target ON company.location_redirect IS 'A rule that redirects to itself is meaningless and would create an infinite-loop risk if the resolver ever re-entered.';
COMMENT ON CONSTRAINT CHK_location_redirect_priority_non_negative ON company.location_redirect IS 'Negative priorities are reserved / undefined.';

COMMENT ON INDEX company.UQ_location_redirect_uuid IS 'Admin CRUD lookups by uuid.';
COMMENT ON INDEX company.IX_location_redirect_source IS 'Primary resolver lookup: find active rules by source_location_uuid. Partial index skips deactivated rules.';
COMMENT ON INDEX company.IX_location_redirect_target IS 'Reverse lookup: which rules currently point at a given location (for admin audit / dedupe).';
COMMENT ON INDEX company.UQ_location_redirect_shape_active IS 'Prevents two active rules with the same (source, range) shape. COALESCE(..., -1) collapses the two NULL columns to a sentinel so NULL-range rules also participate in uniqueness. Does NOT prevent overlapping ranges — that requires a GiST EXCLUDE constraint deferred to a later phase.';

/**********************END SCRIPT******************************/

/******************GRANT PRIVILEGES****************************/

-- Production web service user
GRANT SELECT, INSERT, UPDATE, DELETE ON company.location_redirect TO prwsdbcp;
GRANT USAGE, SELECT ON SEQUENCE company.location_redirect_id_seq TO prwsdbcp;

-- Production analytics user (read-only)
GRANT SELECT ON company.location_redirect TO prbadbcp;

/******************END GRANT PRIVILEGES************************/

/**********************COMMIT**********************************/
COMMIT;
/**********************END COMMIT******************************/
```

### Companion backfill script

The 10+ rules currently hardcoded in `uuidSetter()` migrate in a separate script `DDL_YYYY_MM_DD_LOCATION_REDIRECT_BACKFILL.sql`. Each INSERT should either verify the target UUID exists in `parking_location` (safe path — a failed insert flags a bad rule before production) or accept the FK check that will fire automatically. Rule categories in the backfill:

- **Simple remaps (no range):** Curbstand `c7909083-…` → Beaudry `d8e5a93b-…`.
- **Ranged remaps (majority):** all the "MarinValet → HamptonInnRiverside for 13000-13999" style entries, Safety Park's 8 sub-restaurants, San Juan River Street Marketplace's three range partitions, Josefina/Porzana/SpoonStable overflows, Armature Works/Market Street, SennaHouse ↔ WScottsdale cross-mapping, etc.
- **Unreachable rule NOT migrated:** the `MarinValet && 20249-20751 → Winery` branch in `uuidSetter` is dead code (earlier MarinValet block returns first) — do not backfill; confirm with product whether it should be revived or dropped.

Zero-padding misprints do NOT get a row — they are handled by `sanitizeUuid()` in the resolver.

## Related memories

- `project_uuid_redirects.md` — the specific zero-padding case (family 3). Will remain accurate as an example but will be superseded operationally once the backend resolver ships.
- `guest_page_flow.md` — the wider end-to-end boot; `ticket-info` is the entry point where the remap must happen.
- `project_boot_orchestration.md` — after the FE cleanup, `ticket.component` continues to own the boot chain; only the pre-`ticket-info` remap step is deleted.
