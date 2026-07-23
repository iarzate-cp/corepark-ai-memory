---
name: operatorCompanyId — header pattern (interceptor legacy + exclusion list)
description: Correct place for operatorCompanyId is Operator-Id header; legacy interceptor injects into body but has an EXCLUDED_URLS bypass
type: project
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
The canonical way to send operator context in backoffice service calls is the `Operator-Id` HTTP header (see `TrendsService`, `ReportsCustomOneTimeService`). Do NOT add `operatorCompanyId` to the request body.

**Why:** A legacy `operatorIdentifierInterceptor` still auto-injects `operatorCompanyId` into POST bodies for endpoints not in its exclusion list. When observing outgoing requests (Network tab, curl replay), you will see `operatorCompanyId` in the body — that comes from the interceptor, not from the intended contract.

**How to apply for new endpoints:**
1. In the service, use `Operator-Id` + `Location-Id` headers (mirror `TrendsService`). Do not touch the body for operator context.
2. Add the new endpoint's URL prefix to `EXCLUDED_URLS` in `src/app/core/interceptors/operator-identifier/operator-identifier.interceptor.ts` so the interceptor stops mutating the body. Use the shortest shared prefix (e.g. `/reports/analytics/` covers every analytics endpoint at once).
3. Verify in the Network tab that the outgoing body no longer contains `operatorCompanyId`.

**Existing exclusions (as of 2026-07-22):** `/backoffice/compensations`, `/backoffice/v1/config-parameters`, `/backoffice/guest-profiles`, `/reports/analytics/`.
