---
name: Handoff URLs — defer to /reports/ (or sibling) convention
description: When a backend handoff doc shows a relative-looking path, default to the codebase convention instead of copying the path literally
type: feedback
originSessionId: f07718db-6a66-4d7e-a6be-5f3c2a44ce61
---
When a backend handoff (HTML/PDF/Notion/etc.) gives a path for a new endpoint, do NOT copy it verbatim into `pathSetter(...)`. Handoff authors frequently write the *logical* path within their service (e.g. `POST /valet-analytics-report`), expecting the front to slot it into whatever prefix sibling endpoints already use.

**Why:** Valet Analytics Report (id 18) handoff §3 wrote `POST /valet-analytics-report`. I implemented `pathSetter('/valet-analytics-report')` verbatim → resolved to `https://dev-web-71f618df0c78.corepark.com/valet-analytics-report` and 404'd. Real path is `/reports/valet-analytics-report` — same prefix every other endpoint in `reports-custom-one-time-service.ts` uses. I had even flagged the deviation in my response ("Si el path correcto fuera /reports/valet-analytics-report, dímelo y lo cambio — usé el literal del handoff") but still shipped the deviant version, costing a round-trip.

**How to apply:** Before writing `pathSetter('/<new-path>')`:
1. Look at sibling methods in the same service (or same domain) for the URL prefix convention (`/reports/`, `/billing/`, `/operator/`, etc.).
2. If 100% of siblings share a prefix and the handoff path doesn't, the handoff is almost certainly using shorthand → prepend the convention prefix.
3. Mention the deviation briefly in the response so the user can correct if rare cases really do live outside the prefix, but ship the convention-aligned version by default.
4. Only ask for confirmation when there are *competing* conventions (multiple prefixes used by sibling endpoints) and the handoff is ambiguous.
