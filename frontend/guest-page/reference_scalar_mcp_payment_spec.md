---
name: Reference — Scalar MCP has stale Payment Service spec
description: The Payment Service API doc registered in the Scalar MCP workspace is outdated (title still says "Square, Windcave and FreedomPay gateways", no Stripe endpoints). Don't rely on Scalar MCP for Stripe payment endpoints; use the web UI or a PDF export instead.
type: reference
---

The Scalar MCP transport URL registered locally (`https://api.scalar.com/vector/mcp/0d0afc1e-6280-47ea-a99f-b5b2ea9ead62`) points to a workspace where the **Payment Service API** OpenAPI document is stale.

## Symptoms
- `mcp__scalar__summarize-openapi-specs` returns the Payment Service API entry with:
  - `title: "Payment Service API"`
  - `description: "API documentation for CorePark Payment Service (Square, Windcave and FreedomPay gateways)"` (no Stripe)
  - `paths: []` (empty)
- `mcp__scalar__search-openapi-operations` never returns Stripe endpoints (`/stripe/web/card-on-file`, `/stripe/web/get-parameters`, `/stripe/terminal/*`, etc.) regardless of query.
- Other specs in the same workspace (Valet, PMS, Backoffice, Monolith, Notifications) index correctly with full paths.

## Not fixable by reinstalling the MCP client
The MCP is just a transport — reinstalling it hits the same workspace UUID. The fix must happen on the Scalar side: someone needs to re-upload the current Payment Service OpenAPI spec to that workspace (or the CI pipeline that syncs it needs to run).

## Workarounds for now
- The web UI at `https://api-specs.corepark.com/payment-service-api/tag/stripe/*` **does** show the current Stripe endpoints — the docs website reads from a different source than the MCP workspace.
- To get schemas into the model context: user can screenshot or export a PDF of the relevant endpoint page (that's how the CoF POST `/stripe/web/card-on-file` contract was obtained on 2026-07-23).
- WebFetch of the api-specs URL hits an auth wall (Google sign-in), so it can't be scraped programmatically.

## Before trying Scalar MCP for Stripe endpoints again
Re-run `mcp__scalar__summarize-openapi-specs` and check whether the Payment Service description now includes Stripe. If it still says "Square, Windcave and FreedomPay gateways", the spec hasn't been refreshed and searches will still miss.
