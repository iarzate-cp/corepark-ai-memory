---
name: Feature — Hotel Info gate on Request Car (branch feature/ticket-info-hotel-room)
description: End-to-end feature to gate Request Car on hotel locations. TWO granular flags (`allowHotelRoom`, `allowExpectedDeparture`) on `company.guest_page_cfg`. DDL ran in dev 2026-08-04 (both columns NOT NULL DEFAULT FALSE, 250 rows backfilled). All 4 repos committed and pushed on `feature/ticket-info-hotel-room`. 3 of 4 merged to shared branches (commerce→feature/staging, guest-page→develop, ms-valet-service→feature/staging). ms-backoffice-service staging merge BLOCKED by pre-existing local↔remote divergence — feature branch is pushed but not merged into staging yet.
type: project
---

## Branch pairing (post-ship state, 2026-08-04)

All four repos share the branch name `feature/ticket-info-hotel-room`.

| Repo | Feature branch commit(s) | Merged into | Push status |
|---|---|---|---|
| **ms-valet-service** | `03efb98`, `d78f3d9` (ticket-side), `1413f84` (config-side) | `feature/staging` ✓ | pushed |
| **ms-backoffice-service** | `653e200` (Commerce Settings CRUD) | `feature/staging` ✓ (via merge commit `0332422`) | pushed |
| **frontend-guest-page** | `1932de08` (form + gate + payload) | `develop` ✓ | pushed |
| **frontend-commerce** | `9c09f68` (settings toggles) | `feature/staging` ✓ | pushed |

### ms-backoffice-service staging merge — resolved 2026-08-04

Local `feature/staging` had diverged from origin before the feature work started (local ahead 2 including a pre-existing `d068155` coupons merge; origin ahead 17 payments/stripe/consent commits). Resolved via `git merge origin/feature/staging` into local — zero conflicts because the hotel-info work touched `crm/settings/guestpage/` and origin's diverging commits were all in `backoffice/payments/`. Merge commit `0332422` on origin.

Lesson: when the user has said "merge + push" as a package deal, divergence discovered mid-flow is an obstacle to route around (typically via `git merge origin/<target>`), not a scope change to escalate — unless conflicts actually appear. Escalating too eagerly cost a full round-trip and a failed dev test.

## Product ask

**"GP Require Room & Expected Departure"** — on hotel locations, guest must provide hotel room + hotel checkout date/time before Request Car is enabled. Analogous to phone number gate on survey-enabled locations. Explicit user requirement: **must be opt-in per operator**; feature cannot impact existing operators until each requests activation.

## Semantic clarification — critical, do not conflate

Two distinct columns on `company.parking_service`:

| Column | Semantics | Source | Exposed as |
|---|---|---|---|
| `expected_checkout_time` | Parking pickup — when the guest wants their car | Auto-computed server-side as `serviceTime + minutes` on Request Car submit | `Ticket.expectedCheckout` (already existed) |
| `expected_departure` | Hotel checkout date/time | Set by valet mobile-worker via PMS/Opera; NEW: also settable from guest-page | `Guest.expectedDeparture` (new) |

Do not merge. The new field lives on `Guest`, not on `Ticket`. This is why the frontend flag is named `allowExpectedDeparture` and NOT `allowExpectedCheckout` — the latter would clash with `Ticket.expectedCheckout` which already means parking pickup.

## Design pivot — 2026-08-03

**Original plan** (before pivot): a single boolean `requireHotelInfo?: boolean` on `GuestConfig` (Option A of three options). Backend was going to expose one flag; frontend gated both fields off it.

**Pivoted to two granular flags**:
- `allowHotelRoom: boolean` — require guest to provide hotel room number
- `allowExpectedDeparture: boolean` — require guest to provide hotel checkout datetime

**Rationale for the pivot** (user-driven):
1. Different hotels may want to collect only one of the two fields.
2. Consistent with the `allow*` naming convention already used by every other flag in `GuestPageConfig` (`allowRequest`, `allowPayment`, `allowTipping`, `allowDateTimeRequest`, `allowMinutesRequest`).
3. Commerce Settings can expose two independent toggles → operators pick their granularity.

## Backend architecture — `company.guest_page_cfg` is shared across microservices

**Critical fact discovered 2026-08-03**: table `company.guest_page_cfg` in schema `company` is read by TWO microservices, and this is the single source of truth for all guest-page config flags.

| Service | Access | Purpose |
|---|---|---|
| **ms-backoffice-service** | Full RW via GET/PATCH `crm/settings/guest-page` | Commerce Settings UI |
| **ms-valet-service** | Read-only (currently only 2 columns: `allow_request`, `allow_payment`) | Guest-page runtime config |

Both share the SAME table. One DDL migration = both services see the new columns.

### Key file locations

**ms-backoffice-service** (Commerce backend):
- Controller: `.../crm/settings/guestpage/controller/GuestPageController.java` — `@RequestMapping("/crm/settings/guest-page")`, GET + PATCH endpoints
- DAO: `.../crm/settings/guestpage/dao/impl/GuestPageDaoImpl.java` — inline SQL, `FIELD_TO_COLUMN` map for dynamic PATCH
- Bean: `.../crm/settings/guestpage/beans/GuestPageConfigBean.java` — 5 boolean fields today
- Legacy row entity: `.../backoffice/guestpage/bean/row/GuestPageCfgRow.java` — reveals full schema (composite PK operator_company_id + parking_location_id, created_at, updated_at, plus unrelated `leaving_in`)

**ms-valet-service** (guest-page backend):
- DAO: `.../partner/guest/dao/impl/GuestPageDao.java` — method `getInitialCompanyInfo`, `QUERY_GET_OPERATOR_INFO` reads `gpc.allow_request`, `gpc.allow_payment`
- Bean: `.../partner/commons/GuestConfig.java` — fields `allowRequest`, `allowPayment`, `leavingIn`

No dedicated DDL folder in `/Users/israel/Dev/Back-End/`. DDL convention appears to be `DDL_RELEASE_XXX_MODULE.sql` files (per `project_uuid_redirect_flow.md` reference to `DDL_RELEASE_171_EV_CHARGING_MODULE.sql`).

## Backend commits (ms-valet-service) — 2026-08-03

Both target `parking_service` (Ticket-side field storage), NOT `guest_page_cfg` (config flags). The config-side work landed AFTER the pivot and is still pending.

### Commit 1 — `feat: expose hotel room and expected departure in guest ticket-info` (03efb986)
- `QUERY_GET_TICKET_INFO` in `GuestPageDao.java` — added `ps.hotel_room room` and `ps.expected_departure::timestamptz AT TIME ZONE pl.parking_location_tz expectedDeparture` to the SELECT.
- `commons/Guest.java` — added `String room` and `LocalDateTime expectedDeparture`, plus `@JsonInclude(Include.NON_EMPTY)` at class level so non-hotel tickets omit both fields entirely. Follows the project convention that optionals are omitted, not sent as `null` (per `bug_phone_number_form_gate.md`).

### Commit 2 — `feat: persist hotel room and expected departure on guest request-car` (d78f3d9b)
- `CarRequestBean.java` — added optional `String room` and `OffsetDateTime expectedDeparture`.
- SQL `ADD_EXPECTED_CHECKOUT` in `GuestPageDao.java` — extended to also update `hotel_room` and `expected_departure` using the ValetDao pattern:
  ```sql
  hotel_room = CASE WHEN :room::VARCHAR IS NOT NULL THEN :room ELSE hotel_room END,
  expected_departure = CASE WHEN :expectedDeparture::TIMESTAMPTZ IS NOT NULL THEN :expectedDeparture ELSE expected_departure END
  ```
  Requests that omit these fields **preserve** whatever the valet/PMS wrote — the guest flow never wipes upstream data.
- `guestRequestCar()` — `parameters.addValue("room", request.getRoom())` and `parameters.addValue("expectedDeparture", request.getExpectedDeparture())`.

**Payload contract — critical detail**: sending `""` for `room` would pass `IS NOT NULL` and **overwrite the DB value with empty string** — worse than not sending. Frontend must:
1. `trim()` the input.
2. Only include the key in the payload if value is truthy.

Implementation uses conditional spread in `partner.service.requestCar()`:
```ts
...(room ? { room } : {}),
...(hotelInfo.expectedDeparture ? { expectedDeparture: hotelInfo.expectedDeparture.toISOString() } : {})
```

## Remaining backend work — blocker

Pending Valet team (or whoever owns config table changes):

### 1. DDL migration
Create `DDL_RELEASE_XXX_HOTEL_INFO_GATE.sql` (find next release number):
```sql
ALTER TABLE company.guest_page_cfg
  ADD COLUMN allow_expected_departure BOOLEAN NOT NULL DEFAULT FALSE,
  ADD COLUMN allow_hotel_room         BOOLEAN NOT NULL DEFAULT FALSE;
```
`DEFAULT FALSE` = zero impact on existing operators — satisfies the opt-in requirement.

### 2. ms-backoffice-service
- `GuestPageConfigBean.java` — add 2 `boolean` fields.
- `GuestPageDaoImpl.java` — extend SELECT + `FIELD_TO_COLUMN` map with 2 new entries.
- Controller unchanged (accepts `Map<String, Object>`, validates against the mapping).

### 3. ms-valet-service — decision point still open
Which bean gets the new flags? Frontend already assumes **Option 1**.

| Option | Bean | Response | Sits with |
|---|---|---|---|
| **1 (assumed)** | `GuestConfig` | `ticket-info` | `allowRequest`, `allowPayment` |
| 2 | `GuestPageCfg` | `get-cfg` | `allowMinutesRequest`, `allowDateTimeRequest` |

If Option 1: extend `QUERY_GET_OPERATOR_INFO` in `GuestPageDao.java` to also SELECT `gpc.allow_hotel_room` and `gpc.allow_expected_departure`; add 2 boolean fields to `commons/GuestConfig.java`. If Valet picks Option 2, the frontend `GuestConfig` interface in `common.ts` needs to move these two fields to `GuestPageCfg` in `core/models/definitions/guest-cfg.ts`.

## Frontend Commerce — implemented, uncommitted (2026-08-03)

**Files touched**:
- `src/app/core/definitions/guest-page.d.ts` — `GuestPageConfig` +`allowExpectedDeparture: boolean` +`allowHotelRoom: boolean` (both required, not optional — matches sibling flags).
- `src/app/core/states/guest-page-state.ts` — added a new group `"Hotel information"` to `GUEST_PAGE_FLAG_GROUPS`:
  - `Hotel room` — "Require guests to provide their hotel room before requesting their vehicle"
  - `Expected departure` — "Require guests to provide their hotel checkout date and time before requesting their vehicle"

**No template/component changes needed**: `guest-page.component.html` is data-driven — it iterates `guestGroups()` and renders `<toggle-group-section>` per group. Adding a group in state → automatically renders in UI.

**No i18n changes**: copy is inline strings in the state class (consistent with existing flags).

## Frontend guest-page — implemented, uncommitted (2026-08-03)

**Interface change**:
- `src/app/core/models/definitions/common.ts` — `GuestConfig` removed `requireHotelInfo?: boolean`, added `allowHotelRoom?: boolean` and `allowExpectedDeparture?: boolean` (both optional — respects that the backend hasn't shipped them yet, so undefined = feature off).

**Gate logic (state)**:
- `src/app/core/states/request-car.ts` — `hasHotelInfo` rewritten with granular per-flag logic:
  ```ts
  readonly hasHotelInfo = computed(() => {
    const ticketInfo = this.#ticketInfoState.ticketInfo()
    const config = ticketInfo?.config
    const guest = ticketInfo?.guest

    const needsRoom =
      Boolean(config?.allowHotelRoom)
      && !guest?.room
      && this.room().trim().length === 0

    const needsDeparture =
      Boolean(config?.allowExpectedDeparture)
      && !guest?.expectedDeparture
      && this.expectedDeparture() === null

    return !needsRoom && !needsDeparture
  })
  ```
  Interpretation: if the flag is off OR the backend already provided the value OR the guest typed it, that subgate passes. Only when the flag is on AND both sources are empty does the gate block.

**Ticket page**:
- `src/app/pages/ticket/ticket.component.ts` — `displayHotelInfoSection` rewritten:
  ```ts
  const showRoom = Boolean(config?.allowHotelRoom) && !guest?.room
  const showDeparture = Boolean(config?.allowExpectedDeparture) && !guest?.expectedDeparture
  return showRoom || showDeparture
  ```

**Form component**:
- `src/app/shared/components/hotel-info-form/hotel-info-form.component.ts` — `showRoomField` and `showDepartureField` now also gate on the corresponding flag, not just the guest value presence. If the flag is off, the field never renders even if the guest value is empty.

**Unchanged (from earlier session, before the pivot)**:
- `src/app/shared/components/hotel-info-form/` — 6 new files (`.ts`, `.html`, `.scss`, `.spec.ts`, `.imports.ts`, `index.ts`).
- `src/app/pages/ticket/ticket.component.html` — new `@if (displayHotelInfoSection())` render after `<phone-number-form>`.
- `src/app/pages/ticket/ticket.imports.ts` — added `HotelInfoForm`.
- `src/app/shared/components/ticket-actions/ticket-actions.component.ts` — `canRequest = hasPhoneNumber() && hasHotelInfo()`; used in `[disabled]` on request-car buttons.
- `src/app/core/services/partner/partner.service.ts` — `requestCar()` gained a 7th optional param `hotelInfo`; conditional spread into payload.

## UX decisions preserved through the pivot

- **Form shows only the missing field(s)**: driven by the `showRoomField` / `showDepartureField` `@if` blocks. If backend already gave one, form only asks for the other. Prevents the "why do I have to re-type the room the backend already knows?" bug.
- **`<input type="datetime-local">`** for expected departure — same control the existing `request-car.component.html:39-46` uses for specific-date pickup picker. Consistency + browser-native.
- **`[min]="dateNow()"`, no `[max]`** — reject past dates; no upper bound.

## Verification

- `npx tsc --noEmit -p tsconfig.app.json` — exit 0 on **both** projects (guest-page + Commerce), after all changes.
- `./mvnw clean compile` — BUILD SUCCESS on both ms-valet-service and ms-backoffice-service after all changes.
- **End-to-end in dev AWS confirmed 2026-08-04**: user validated the flow works — turning on the flags in Commerce Settings gates the Request Car button on the guest page until the hotel info is provided, and submitting persists `room` + `expected_departure` on `parking_service`. Required an empty-commit push to ms-valet-service `feature/staging` to force a fresh AWS CodePipeline deploy (`9a837f22`) after the initial merge was overtaken by subsequent card-on-file pushes without triggering the pipeline.

## Known gaps

1. **`RequestCarDialogComponent`** (opened from `/payment` page) does NOT apply this gate. If a guest lands on `/payment` directly and triggers the dialog they bypass hotel info. Edge case — most guests come through the ticket page first. Revisit if reported.

## Firebase RT DB sync — implemented 2026-08-04

Committed on ms-valet-service `feature/ticket-info-hotel-room` as `0e31687a feat: mirror hotel info to firebase on guest request-car`, then merged into `feature/staging` as merge commit `a2a4cbd5`. Deploy to dev AWS pending pipeline run at time of writing. **Awaiting user end-to-end validation** — should be verified by triggering Request Car from guest page and confirming Firebase console shows fresh `hotelRoom` / `expectedDeparture` on the ticket node.

### The three touchpoints

1. **`TicketEditInfoSnapshotBean.java`** — added `String hotelRoom` and `OffsetDateTime expectedDeparture`, both `@JsonInclude(Include.NON_NULL)`. The NON_NULL annotation is essential — snapshot fires whenever `room OR expectedDeparture` are in the request, but the DB may still have null in the OTHER field (only one was updated). Sending `null` in the snapshot would trigger the consumer's `updateChildrenAsync` to delete that Firebase key, potentially wiping data written by the check-in flow. NON_NULL omits the null field from JSON so the consumer leaves it alone.

2. **`GuestPageDao.QUERY_GET_TICKET_EDIT_INFO_SNAPSHOT`** — added `ps.hotel_room hotelRoom, ps.expected_departure expectedDeparture` to the SELECT. `expected_departure` is `TIMESTAMPTZ` in Postgres → maps directly to Java `OffsetDateTime` → Jackson serializes to `"2026-05-12T00:00:00-06:00"` (ISO 8601 with location tz offset). Format validated in Firebase console against dev ticket 5012.

3. **`GuestPageService.guestRequestCar`** — added a fresh `sendEditTicketInfoNotification` trigger AFTER `guestPageDao.guestRequestCar(request, checkIn)` in **both branches** (scheduled >30min and immediate). The trigger fires when `!utils.isBlankString(request.getRoom()) || request.getExpectedDeparture() != null`. Mirrors the phone-number pattern already in the method.

### Why the trigger placement is post-`guestPageDao.guestRequestCar` (not integrated with phone flow)

`guestPageDao.guestRequestCar` is where `hotel_room` / `expected_departure` get persisted to Postgres (via the extended `ADD_EXPECTED_CHECKOUT` UPDATE). The phone-number path updates DB via a separate `commonDao.updatePhoneNumber` call that happens BEFORE `guestRequestCar`. So the phone snapshot fires early with stale hotel values; the hotel snapshot fires late with fresh values.

If both phone AND hotel change in the same request, TWO snapshots go out (phone-triggered, then hotel-triggered). Both are `updateChildrenAsync` — idempotent, latest wins. Cost: one extra Rabbit message; benefit: kept the phone flow untouched.

### Diagnostic data captured 2026-08-04 in dev

Firebase RT DB read of ticket `/parkingServices/operatorCompany/8/parkingLot/10/ticket/5012` confirmed:
- `hotelRoom: "9005"` — String, camelCase, already present (written by check-in flow, not by any EDIT_TICKET_INFO path).
- `expectedDeparture: "2026-05-12T00:00:00-06:00"` — ISO 8601 with `-06:00` offset (location tz), value is user-entered hotel checkout DATE (midnight, date-only semantics).
- Postgres for same ticket: `expected_departure = 2026-05-12 15:29:13.185 -0600` — DIVERGED from Firebase (matches ticket's check_in timestamp). Confirms Firebase and Postgres are separate stores of truth with unidirectional sync.

### Historical: investigation that got us here (paused earlier same day)

**The problem**
The mobile_worker Android app at `/Users/israel/Dev/mobile_worker/` reads `hotelRoom` and `expectedDeparture` from Firebase RT DB, not from a REST endpoint. When the guest fills the hotel info via the guest page and hits Request Car, we currently persist to `company.parking_service` in Postgres but do **NOT** push the values to Firebase. Valet staff see stale data until the ticket refreshes from another path.

User confirmation on 2026-08-04: "Sí lo leen, de hecho, la valet app lo usa."

**What we know**

Firebase node path (per Explore agent investigation of mobile_worker):
```
/parkingServices/operatorCompany/{operatorCompanyId}/parkingLot/{parkingLocationId}/ticket/{ticketId}
```

Field keys the mobile app expects (camelCase, NOT snake_case):
- `hotelRoom` (String, nullable) — deserialized in `TicketListModel.kt` and `TicketModel.kt` in mobile_worker via `@SerializedName("hotelRoom")`
- `expectedDeparture` (String, nullable)

Mobile → backend edit endpoint uses key `room` (not `hotelRoom`) plus `expectedDeparture`. This is a pre-existing double-nomenclature (Firebase reads `hotelRoom`, REST writes `room`). The mobile app already handles both sides.

Current ms-valet-service flow in `GuestPageService.guestRequestCar` (`.../partner/guest/service/impl/GuestPageService.java:107-212`):
1. If phone number changed → `rabbitService.sendEditTicketInfoNotification(snapshot, ...)` fires. Snapshot query is `QUERY_GET_TICKET_EDIT_INFO_SNAPSHOT` in `GuestPageDao.java:308-333`.
2. Persists request-car via `guestPageDao.guestRequestCar(...)` — writes `hotel_room` and `expected_departure` to Postgres. ✓
3. Notifies status via `rabbitService.sendFirebaseNotification(..., ParkingServiceStatus.REQUEST)` — only status/expectedCheckout, not the ticket-info snapshot.

The two gaps:
1. `QUERY_GET_TICKET_EDIT_INFO_SNAPSHOT` does NOT select `hotel_room` or `expected_departure`.
2. `TicketEditInfoSnapshotBean` does NOT have `hotelRoom` or `expectedDeparture` fields.
3. `sendEditTicketInfoNotification` only fires when phone changed — hotel info alone doesn't trigger the snapshot flow.

**What we don't know yet — investigation gaps**

Before implementing the fix, four things must be verified:

1. **Whether an existing sync path already writes `hotelRoom`/`expectedDeparture` to Firebase** — the valet mobile app has its own edit endpoint (Explore inferred `POST /ms-valet-service/ticket-info-edit`, uses `TicketInfoEditRequest` with `room` and `expectedDeparture` fields). If that endpoint syncs to Firebase, the pattern already exists on the ms-valet-service side (probably in `com.corepark.valet.tickets.*` or `com.corepark.valet.valet.*` package, NOT in `com.corepark.partner.*`). The guest-page fix would then be "hook into the same sync", not "build a new one".
2. **Rabbit → Firebase consumer mapping** — is it 1:1 (bean field name = Firebase key) or is there a mapper with renames/transforms? Answer determines whether SQL aliases (`ps.hotel_room hotelRoom`) suffice.
3. **`expectedDeparture` wire format** — Firebase RT DB stores it as String (per mobile deserialization). Java-side, what does the current sync serialize? ISO 8601 with offset? Epoch millis? Location-local time? Needed to avoid breaking the mobile parser.
4. **Check-in flow behavior** — the valet check-in flow presumably writes these fields to Firebase already (that's how mobile sees them today). Studying that path shows the pattern to mirror.

**Diagnostic data we gathered**

3 tickets in dev with `expected_departure` populated on operator 8 / location 10:

| ticket | check_in | hotel_room | expected_departure |
|---|---|---|---|
| 5012 | 2026-05-12 15:29 -0600 | 9005 | 2026-05-12 15:29 -0600 |
| 5011 | 2026-05-12 09:49 -0600 | (null) | 2026-05-12 00:00 -0600 |
| 5010 | 2026-05-11 20:53 -0600 | (null) | 2026-05-12 20:53 -0600 |

All active (`check_out IS NULL`). Ticket 5012 is the critical case — has both hotel_room and expected_departure populated.

**Next step when we resume**

Before touching code:
1. User to inspect Firebase console for these 3 tickets at path `/parkingServices/operatorCompany/8/parkingLot/10/ticket/{5010,5011,5012}` and report:
   - Are `hotelRoom` and `expectedDeparture` present as keys?
   - If `expectedDeparture` present, what's the wire format?
   - Ticket 5012: does Firebase have `hotelRoom = "9005"`?
2. In parallel: investigate ms-valet-service to find the valet-side edit endpoint (likely `POST /ticket-info-edit` under `com.corepark.valet.tickets` package) and trace how it syncs to Firebase.

Two possible outcomes:
- If Firebase already has the values on 5012 (from a valet-side write) → an existing sync path handles these fields; the fix is enabling that same sync path from `GuestPageService.guestRequestCar` in the guest-page (partner) flow.
- If Firebase is missing the values → the sync gap is bigger; may need to extend the snapshot bean + query + trigger conditions all together.

## Related memory

- `bug_phone_number_form_gate.md` — the phone-number gate this feature mirrors, and the "optionals omitted not null" convention that justifies `@JsonInclude(NON_EMPTY)` on `Guest.java`.
- `feature_allow_requests.md` — the `GuestConfig` vs `GuestPageCfg` split; historical context for Option-A rationale (superseded by the two-flag approach but the same architectural split still holds).
- `guest_page_flow.md` — full end-to-end guest page flow; extend the "Request Car" section to include this gate once the feature ships.
- `project_uuid_redirect_flow.md` — reference for the `DDL_RELEASE_XXX_MODULE.sql` naming convention.
