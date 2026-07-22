---
name: Project Overview
description: What this app is, its tech stack, environments, and where docs live
type: project
originSessionId: 1a79ed89-b142-4110-a666-1033820e8843
---
Angular 21 valet parking operator web app (standalone + signals-first). Staff manage vehicle tickets, check-ins/outs, payments, settings per parking location.

**Stack:** Angular 21, Angular Material 21, Firebase Realtime DB (`@angular/fire@20`), JWT auth (Bearer), `@ngx-translate` (en/es), ng2-charts, Luxon, CryptoJS, pnpm 10, `@corepark/corepark-ui` (Corepark design system).

**Environments:**
- Dev API: `https://dev-web-71f618df0c78.corepark.com`
- Prod API: `https://api-web.corepark.com`
- Dev Firebase: `corepark-services-dev.firebaseio.com`
- Prod Firebase: `corepark-services.firebaseio.com`

**Docs:** `/Users/israel/Dev/frontend-valet-web/docs/` — created 2026-05-12
- `overview.md` — purpose, stack, features
- `architecture.md` — folder structure, path aliases, conventions
- `routing.md` — route tree, guards, auth flow
- `services.md` — all HTTP services + known bugs
- `states.md` — all signal-based states
- `components.md` — all shared components
- `http-api.md` — endpoint table, interceptor pipeline
- `firebase.md` — Firebase setup, known bug workaround
- `i18n.md` — translation key structure

**Why:** Full project analysis done 2026-05-12 to establish a knowledge baseline.
**How to apply:** Reference docs before adding new features to follow existing patterns.
