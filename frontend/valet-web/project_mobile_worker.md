---
name: Mobile Worker Reference
description: mobile_worker is the Android source app; frontend-valet-web is its web distillation. Key facts for bridging both.
type: project
originSessionId: 1a79ed89-b142-4110-a666-1033820e8843
---
The web project (`frontend-valet-web`) is a browser-based version of the CorePark Valet Android app at `/Users/israel/Dev/mobile_worker`.

**Why:** The web app is a subset of the Android app — same backend API, same domain, fewer features. When adding new features to the web app, look at the mobile implementation first for the correct API contract.

**How to apply:** Use `docs/mobile-worker-reference.md` (in this web project's docs/) as the authoritative reference for API endpoints, request/response shapes, ticket status codes, payment methods, Firebase paths, and feature parity table.

## Key facts

- Mobile base URLs differ from web base URLs (different subdomain) but both hit the same backend API.
  - Mobile dev: `https://dev-mobile-ebc4fdb639b1.corepark.com`
  - Web dev: `https://dev-web-71f618df0c78.corepark.com`
- Auth is two-step: location login (`/security/oauth/token`, Basic auth, form-encoded) → valet PIN (`/credentials/login-pincode`, Bearer).
- All location-scoped requests pass `operatorCompanyId` + `parkingLocationId` — either as body fields or `Operator-Id` / `Location-Id` headers.
- Ticket status wire values use spaces/hyphens, not underscores (e.g. `"CHECK-IN"`, `"TEMPORAL CHECKOUT"`).
- Payment method wire codes: CASH=`"C"`, CARD=`"B"`, POST_TO_ROOM=`"P"`, PARKING_AGGREGATOR=`"PA"`.
- Firebase paths: `/parkingServices/operatorCompany/{id}/parkingLot/{id}/ticket` for real-time ticket updates.
- Features not yet in web: Park Car, Relocate, Request Manager, Compensations, Clock In/Out, Daily Report, Cash Control, Overnight flows, Refunds, Reinstate, Broadcast Messages, all hardware integrations.
