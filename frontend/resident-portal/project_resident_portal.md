---
name: Resident Guest Pass Portal
description: CorePark spec v1.2 for residential guest parking portal — Israel owns frontend (Resident Portal Angular 21 app + Back Office additions)
type: project
originSessionId: c06f3dcc-4017-48c0-807f-f6ff48788f85
---
Self-service guest parking purchase, QR delivery, and redemption — white-labeled per residential property.

**Why:** Residential properties (apartments, condos, HOAs) need a self-service product to offer pre-paid guest parking. Currently handled manually or not offered, blocking a revenue stream for CorePark's residential operators.

**Jira:** PM-RESIDENTIAL (to be created). Spec v1.2 prepared by Elias Hassan for Israel Arzate.

**Effort:** ~60 points · 6–8 weeks across team.

**How to apply:** v1.2 is the source of truth. It overrides v1.0 and v1.1 wherever they conflict.

---

## Ownership

- **Israel** — Resident Portal (new Angular 21 app) + Back Office additions (Angular)
- **Jorge** — Backend (Spring Boot 3.5 / Java 17)
- **Oscar** — Database (PostgreSQL, Flyway migrations, pool seeding)
- **Eli** — Testing (end-to-end) + email HTML templates (follow-up)

## What Israel builds

### 1. Resident Portal (new Angular 21 standalone app)
- URL: `residential.corepark.com/{operator}/{location}` · Staging: `residential-staging.corepark.com/{operator}/{location}`
- **4 routes** (v1.2 added /recover): landing, checkout, passes, recover
- Portal is **open to anyone** — no auth, no login, no OTP. Payment is the gate.
- Branding via CSS custom properties set from `residential_location_config` on route resolve
- QR codes rendered with `angularx-qrcode`; payload = `code_uuid` string
- Post-payment: poll `GET /purchases/{id}/codes` every 1.5s up to 20s (202→keep polling, 200→render, 409→refund msg, 410→redirect to /recover)
- SMS link to `/passes/{purchaseId}?t={token}` — token is HMAC-SHA256, 7-day expiry
- `/recover` route: email + phone form → `POST /api/v1/residential/recover` → always same success message
- Deploy as static SPA to S3/CloudFront (not yet provisioned — Israel files DevOps ticket after Phase 0 spike)
- Mobile-first: single column below 640px, 44px tap targets, no horizontal scroll, QR min 220px, test at 375px

### 2. Back Office additions (existing Angular app)
- New "Residential Bundle" rate type in the Rates module (3 fields: quantity, price, display_name)
- New "Residential Passes" dashboard: Pool status card + Sales table + Redemptions table
  - Reuse BO-327/BO-328 table primitives — do NOT build new table components
  - Sales table has CSV export (v1); redemptions export is v2
  - Dashboard requires `residential_passes.view` permission (BO-333 RBAC)
- New "Residential Portal" tab in location Settings:
  - slug, display name, logo (plain URL field — file upload is a future swap), primary color, accent color
  - pool alert threshold, deliver email toggle, deliver SMS toggle, portal enabled toggle
  - **support_email** field (new in v1.2 — used as reply-to on resident emails)

## Key API endpoints (Israel consumes)

- `GET /api/v1/residential/locations/{slug}` — PUBLIC, loads branding + bundles
- `POST /api/v1/residential/purchases` — PUBLIC, no auth required
- `GET /api/v1/residential/purchases/{id}/codes?t={token}` — PUBLIC token-gated, returns 202/200/409/410
- `POST /api/v1/residential/recover` — PUBLIC, always returns 202
- Admin endpoints under `/api/v1/residential/admin/locations/{id}/...` — JWT auth

## v1.2 decisions (all pre-sprint blockers resolved)

| Topic | Decision |
|---|---|
| Auth | No auth — portal open to anyone with URL |
| Pass recovery | /recover route + POST /recover endpoint (sends to on-file phone only) |
| FreedomPay | Phase 0 spike first (1 day), then implement |
| Polling | 202/200/409/410 on GET /codes, poll 1.5s up to 20s |
| Discount | Full daily-rate validation (one pass = one free day) |
| Logo upload | Plain URL field now; file upload component swapped in later |
| Email | Two emails: order confirmation (immediate) + PDF (after allocation) |
| Email sender | "CorePark on behalf of {property name}", reply-to = support_email |
| Mobile | Mobile-first, no wireframes, design tokens only |
| RBAC | residential_passes.view permission, tied to BO-333 |
| CSV export | Sales table only (v1), using existing master-report pattern |
| Infrastructure | Israel files DevOps ticket for S3/CloudFront after Phase 0 spike |
| Abandoned purchases | AbandonedPurchaseSweeper marks INITIATED > 30min as ABANDONED |

## Schema changes v1.0 → v1.2

- `residential_purchase.status` enum: added `ABANDONED`
- `residential_purchase`: added `abandoned_at` column + partial index on `(status, created_at)`
- `residential_location_config`: added `support_email VARCHAR(255)` column

## Out of scope (v1)
Login · PMS folio posting · refund flows in UI · code expiration · Valet App changes · per-pass sharing/forwarding · Windcave/Square adapters · redemptions CSV export · auto pool seeding

## Phases (Israel's work)
- **Phase 0** (Israel): FreedomPay spike — 1 day, output is Confluence page + effort estimate
- **Phase 3** (Israel): Resident Portal Angular 21 app
- **Phase 4** (Israel): Back Office additions
- Phase 1 (Oscar): DB migration
- Phase 2 (Jorge): Backend services
- Phase 5 (cross-fn): Reporting
