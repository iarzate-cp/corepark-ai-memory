---
name: The user builds features end-to-end — never assume separate backend/frontend owners
description: Israel builds both backend and frontend for the features he works on. Names of other engineers in specs or docs are planning artifacts, not actual owners. Never route work to third parties or gate on someone else's delivery.
type: feedback
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
**Rule:** Israel is the sole developer on the features he brings to sessions. Backend, frontend, everything — he owns it E2E. When a spec lists names next to backend tasks (e.g., "Oscar" or "Jorge" as owners), **ignore those assignments** and assume Israel does everything.

**Why:** In the coupons enhancement session (2026-07-09), I kept referencing "Oscar and Jorge handle backend" from a task-breakdown table in the enhancement PDF. After Israel corrected me once ("yo hice el desarrollo E2E"), I still hedged with "you could respect the spec's assignment or take backend into your own hands." He had to correct me a second time: "sigo sin entender por qué te cuesta entender que yo hice todo." The mistake burned trust and time.

**How to apply:**

- When a spec assigns tasks to multiple people, treat the assignments as **planning artifacts only** — do not gate work on "someone else's delivery".
- Do not offer options like "wait for backend team" or "we could split with X" — Israel is doing all of it.
- When mapping blockers/dependencies, phrase them as **things Israel needs to build**, not as things blocked on others.
- If the spec mentions "backend endpoint doesn't exist yet", the answer is "you build it in the backend repo", not "someone else will hand it to you".
- The relevant backend repo for corepark features is typically `ms-backoffice-service` (see reference_backend_microservices.md and reference_backoffice_backend_claudemd.md). Israel touches it directly.
- If for some reason another engineer really is involved, Israel will say so explicitly. Absent that, single-owner assumption.

**Never do this in a response:**
- "Oscar handles X, Jorge handles Y..." — even if a doc says so
- "You could wait for backend team..." — there is no backend team; there is Israel
- "Depends on whether backend has task N done" — the answer is "you decide when to do task N"

This applies to future features too — the coupons spec is one instance of a pattern.
