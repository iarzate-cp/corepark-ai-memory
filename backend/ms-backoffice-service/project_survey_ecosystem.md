---
name: Survey feature — three-repo ecosystem and persona split
description: How survey functionality is split across ms-backoffice-service (backend), frontend-survey (end-user SPA), and frontend-commerce (admin CRUD to build)
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
The survey feature spans **three repos** with distinct responsibilities and personas.

| Repo | Persona | Role | Status (2026-07-01) |
|---|---|---|---|
| `ms-backoffice-service` (Java) | Both | Backend for both flows | Only end-user endpoints exist; admin CRUD to be built |
| `/Users/israel/Dev/frontend-survey` (Angular 17) | Parking guest | Answers a survey via link/QR | In production, do NOT modify without explicit approval |
| `/Users/israel/Dev/frontend-commerce` (Angular 21) | Operator admin | CRUD to manage survey templates | Zero survey code — to be built |

**Why:** Confusing the two personas leads to the wrong endpoint contracts. The end-user flow is petition-scoped (URL carries `/{operatorId}/{locationId}/{petitionUUID}`, submit uses those 3 UUIDs). The admin CRUD is template-scoped (identify by `surveyUUID` for read/update/delete; scope by `operator+location` for list/create).

**How to apply:** Whenever a survey-related task lands, first ask which persona it serves. Do NOT overload the existing `/survey/questionnaire` endpoints — they have a real production consumer (`frontend-survey`). Admin CRUD must go under a new namespace (working proposal: `/backoffice/v1/surveys`).
