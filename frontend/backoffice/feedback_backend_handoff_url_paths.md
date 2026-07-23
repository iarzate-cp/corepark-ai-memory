---
name: Handoff URLs — ask, don't assume the prefix
description: When a backend handoff shows a relative path, look at siblings AND ask if unsure — the /reports/ prefix has burned us twice
type: feedback
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
When a backend handoff (HTML/PDF/Notion/etc.) gives a path for a new endpoint, do NOT copy it verbatim into `pathSetter(...)`, and DO NOT guess. Look at sibling endpoints in the same domain, and if there's any ambiguity, ask the user.

**Why:**
- **Incident 1 (2025-11-something):** Valet Analytics Report handoff wrote `POST /valet-analytics-report`. I used it verbatim → 404. Real path was `/reports/valet-analytics-report`.
- **Incident 2 (2026-07-22):** Activity by Rate Class handoff wrote `POST /analytics/tickets/parking-location/activity`. Siblings in `employee-center.ts` and `locations.ts` used `/backoffice/*`, so I picked `/backoffice/analytics/...`. Wrong — the correct prefix for reports-service endpoints is `/reports/*`. User had to correct me.

**How to apply:**
1. **Never copy verbatim.** Handoff authors write logical paths relative to their microservice, expecting the front to slot in the prefix.
2. **Check siblings in the SAME microservice first.** If the artifact says `MS-REPORTS-SERVICE`, look at `trends.ts` and `dashboard.ts` (both use `/reports/*`) — not at `employee-center.ts` or `locations.ts` which use `/backoffice/*`. Different microservices, different prefixes.
3. **If the microservice isn't obvious from the artifact, ASK.** The 30 seconds to confirm the prefix beats a round-trip with a 404.
4. Mention the deviation in the response so the user can correct: "El artifact dice `/X/...` pero uso `/reports/X/...` por convención de sibling endpoints — dime si es otro prefijo."
