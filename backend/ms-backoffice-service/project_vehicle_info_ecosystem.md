---
name: Vehicle Info Links & Guest Vehicle-Info Edit — feature ecosystem
description: Five-repo feature (guest self-service + admin lookup) SHIPPED to shared dev branches on 2026-07-17. Endpoints, envelopes, design decisions, staging commits, PROD deploy still pending
type: project
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
# Vehicle Info Links & Guest Vehicle-Info Edit

Fact: single feature spanning **five repositories**, two flows: a public guest self-service page for missing vehicle info, and an admin tool to copy/export the guest-page URL for any active ticket at a location.

**Why:** valet team often can't complete plate/maker/model/color on check-in under load. Sending the URL to the guest (or letting a client rep share it) shifts the data entry to the guest and reduces wrong-plate risk at pickup. Some clients (IAAS-like partners) don't want to send the SMS themselves — they want the URL to share via their own channels.

**How to apply:** when work touches this feature, remember it's *five repos on a shared branch each* — coordinated deploys are the norm. Don't split the wire contract into a separate PR from its consumer.

## The five repos and their branches

| Repo | Feature branch | Note |
|---|---|---|
| `ms-valet-service` | `feature/guest-vehicle-info-edit` | Owns `GET/PUT /valet/vehicle-info/{locationUuid}/{ticket}` (public); uses **legacy** `Response<T>` + `MessageCode` — that's the valet-package convention |
| `ms-gateway-service` | `feature/guest-vehicle-info-edit` | CORS + `permitAll` for `/valet/vehicle-info/**` |
| `ms-backoffice-service` | `feature/vehicle-info-links` | Owns `GET /backoffice/vehicle-info-tickets` (auth); uses **modern** `ApiResponse<T>` + `ErrorCode` per CLAUDE.md |
| `frontend-guest-page` | `feature/guest-vehicle-info-edit` | New route `/vehicle-info/:locationUUID/:ticket` + `VehicleInfoLayout` |
| `frontend-commerce` | `feature/vehicle-info-links` | New admin page `/settings/vehicle-info-links` under SETTINGS |

## Endpoints (wire contract)

**Guest — `ms-valet-service`:**
- `GET  /valet/vehicle-info/{locationUuid}/{ticket}` → `Response<VehicleInfoPayload>`
- `PUT  /valet/vehicle-info/{locationUuid}/{ticket}` → `Response<VehicleInfoPayload>` (returns updated payload)

`VehicleInfoPayload`:
```json
{
  "operatorUuid": "…",
  "vehicleInfo": { "maker": "…", "model": "…", "color": "…", "plate": "…" },
  "guest":       { "firstName": "…", "lastName": "…", "phoneCode": "+1", "phoneNumber": "5551234567" }
}
```

New MessageCodes: `BUSINESS_LOCATION_NOT_FOUND` (404), `BUSINESS_TICKET_NOT_ACTIVE` (404).

**Admin — `ms-backoffice-service`:**
- `GET /backoffice/vehicle-info-tickets` — headers `Operator-Id`, `Location-Id` (both `@Min(1)`) → `ApiResponse<List<VehicleInfoTicketRow>>`

Row (with `complete` computed in SQL, `locationUuid` used by the FE to build the guest-page URL):
```json
{
  "ticket": 12345, "checkIn": "…", "firstName": "…", "lastName": "…",
  "maker": "…", "model": "…", "color": "…", "plate": "…",
  "complete": true, "locationUuid": "…"
}
```

Query joins `parking_service + parking_location + LEFT JOIN cat_country`, filters `check_out IS NULL`, orders by `check_in DESC`.

## Non-obvious design decisions (verify before changing)

- **All four vehicle fields required** (both BE `@NotBlank` and FE `Validators.required`) — the valet team physically verifies the plate at pickup, so partial submits are worse than none.
- **Phone comes as split `phoneCode` + `phoneNumber`** on the wire (BE joins `cat_country` to split) — parsing a concatenated string on the client was ambiguous.
- **Firebase sync via RabbitMQ with HTTP fallback to `ms-firebase-service`** — DB is source of truth; Firebase push is best-effort and never blocks the response.
- **URL host is env-driven** — `environment.guestPageHost` in each env file (localhost:4100 / CloudFront / guest.corepark.com). See `reference_env_driven_url_host.md` for the reusable pattern.
- **`operator_company` is the root** — see `project_operator_location_invariant.md`. Queries starting from a location can trust `operatorUuid` is not null.
- **Service layer throws domain exceptions** — `GuestVehicleInfoService` uses `.orElseThrow(new ServiceException(MessageCode.X))`; no hand-catching of `DataAccessException`. Backoffice slice has no negative paths, so no throws needed. See `feedback_prefer_throw_over_return_error_envelope.md` for the ecosystem-wide preference.

## Testing

- `ms-valet-service`: JUnit 5 + Mockito, **45 green** (bean validation 16 + controller 6 + service 11 + DAO 12). Service tests use `assertThrows(ServiceException.class)` after the exception-based refactor. Recall: valet uses JUnit 5, backoffice uses JUnit 4 (see `project_survey_testing_conventions.md`).
- `ms-backoffice-service`: JUnit 4 + Mockito, **8 green** (controller 2, service 2, DAO 4).
- **Backend total: 53 green.**
- `ms-gateway-service` did not get new unit tests — changes were configuration-only (CORS whitelist + one `permitAll` matcher) with no business logic. Follows the CLAUDE.md convention (cover DAOs and services; integration covers glue).

## Deploy state (as of 2026-07-17 EOD — closed for the day)

**All 5 repos merged & pushed to their shared dev deploy branch.** Feature is on dev; PROD deploy pending until staging validation.

| Repo | Deploy branch | HEAD |
|---|---|---|
| `ms-valet-service` | `feature/staging` | `7b119c2d` (ServiceException refactor merged) |
| `ms-gateway-service` | `feature/staging` | `c2512b2` (CORS whitelist union) |
| `ms-backoffice-service` | `feature/staging` | `33f7c5a` |
| `frontend-commerce` | `feature/staging` | `3a601f4` (cp-table + cp-badge redesign merged) |
| `frontend-guest-page` | `develop` | `478e3262` (odd repo out — see `reference_guest_page_deploy_branch.md`) |

**Next step** = open PRs to `main` in each repo when the team is ready to promote to PROD.

## Merge conflicts on `feature/staging` (resolved by union)

- `ms-gateway-service` — CORS whitelist: kept `http://localhost:4400`, `http://localhost:4200`, `https://localhost:4400`, `https://localhost:4100`. Dev-only whitelist, so extra entries are harmless.
- `frontend-commerce` — 7 files, all union merges with the newly-landed coupons + rates work: `environment{,.dev,.prod}.ts` (kept `googleKey` + `guestPageHost`), `tsconfig.app.json` (types `["google.maps", "node"]`), `path-service-setter.ts` (kept RATES + RATE_SEQUENCES + COMPENSATIONS + VALET_APP_SETTINGS alongside VEHICLE_INFO_TICKETS), `app.routes.ts`, `main-layout.component.ts`. `pnpm install` had to run after merge to pull `@googlemaps/js-api-loader` into the lockfile.

## Full write-up

`~/Dev/Features/2026_07_17_vehicle_info_links.md` — Jira-ticket-shaped MD with acceptance criteria, wire contracts, testing, follow-ups. Part of the `~/Dev/Features` archive (see `reference_features_archive.md`).
