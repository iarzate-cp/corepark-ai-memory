---
name: Coupon feature — two-repo ecosystem
description: Bulk coupon/voucher generation feature. Admin persona operates from frontend-commerce; backend lives in ms-backoffice-service. Voucher consumption (USED transition) presumably happens elsewhere (ms-valet-service) but is out of scope for this feature
type: project
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
Feature entered analysis phase on 2026-07-07. Origin: manual `PL_COUPON_BULK_INSERT.sql` script that the team has been running by hand to provision coupon batches per parking location. The goal is to expose the same behavior through an admin UI in `frontend-commerce`.

## Repos in scope

| Repo | Role | Persona | Status (2026-07-07) |
|---|---|---|---|
| `ms-backoffice-service` (Java) | Backend that exposes the bulk insert + reads + eventual edit/delete | Operator admin | To be built |
| `/Users/israel/Dev/frontend-commerce` (Angular 21) | Admin UI: create batches, list coupons per location, download Excel | Operator admin | To be built |

## Repos out of scope

- `ms-valet-service` (or whichever service consumes vouchers) — this is where `coupon_voucher.status` transitions from `ACTIVE` to `USED`. Not touched by this feature.
- `frontend-survey` — unrelated.

## Personas

- **Operator admin** (single persona): uses the CRUD to create batches of vouchers for their partners (hotels, restaurants) and hands out the printed/emailed vouchers to end customers. Never sees the customer flow.
- Customer-facing consumption is fully **out of scope** — this feature only manages the *supply* of vouchers.

## Analogy to survey

Structurally the closest peer is the survey admin CRUD (`com.corepark.backoffice.survey`):
- Same shape: template + child rows generated per template (survey → questions → options / coupon → vouchers).
- Same tenancy: `(operator_company_id, parking_location_id)` on every table.
- Same UX shape: admin CRUD from `/settings/*` in frontend-commerce.

Follow the same conventions (bean/ subtree, ApiResponse<T> envelope, ErrorCode enum, JUnit 4 + Mockito tests, springdoc-openapi with `<Feature>ApiExamples`) — see `project_claude_md_source_of_truth.md` and `reference_api_response_pattern.md`.
