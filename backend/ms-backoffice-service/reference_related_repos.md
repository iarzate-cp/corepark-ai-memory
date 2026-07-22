---
name: Related repos and memory scopes for cross-repo work
description: Paths to frontend-survey, frontend-commerce, and design-system; their memory scopes; important cross-repo dependencies
type: reference
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
Features on `ms-backoffice-service` often span sibling repos. Key paths:

- **`/Users/israel/Dev/frontend-survey`** — Angular 17.3 end-user SPA (parking guest answers survey via link/QR). Runs on port 4100 (`npm start`, HTTPS). Env endpoint: `https://dev-web-gateway-corepark.ngrok.io`. No dedicated memory scope; read source directly.
- **`/Users/israel/Dev/frontend-commerce`** — Angular 21.2 admin dashboard (operator/back-office UI). Version 3.4.2 as of 2026-07-01. Uses `@corepark/corepark-ui@0.0.22` (bumped from 0.0.13 on branch `feature/survey-admin-crud`). **Memory scope** at `~/.claude/projects/-Users-israel-Dev-frontend-commerce/memory/` — full architecture map + WIP notes.
- **`/Users/israel/Dev/design-system`** — Angular 21.2 workspace producing the `@corepark/corepark-ui` npm package. Current version `0.0.22` on branch `develop`. Consumed by commerce, backoffice, validation, valet-web. Local dev sync via `pnpm run sync:commerce` (rsync `dist/corepark-ui/` → consumer's `node_modules`). **Memory scope** at `~/.claude/projects/-Users-israel-Dev-design-system/memory/` — component API surface, token system, build pipeline.
- **`ms-backoffice-service`** (this project) — Spring Boot 3.5, Java 17 target. Backend for all frontends.

**Cross-repo constraints:**
- **Do NOT modify `frontend-survey`** without explicit approval. It's in production and serves customer submissions. If a change forces frontend-survey to adapt, coordinate separately.
- **Do NOT commit to `design-system`** from ms-backoffice sessions (feedback rule from Israel — see design-system memory `feedback_commits.md`). Consume via `@corepark/corepark-ui` and rely on the design-system team to publish versions.
- **`@corepark/corepark-ui` imports:** always `@corepark/corepark-ui` (root barrel), never subpaths like `corepark-ui/components`. Subpath exports aren't configured in the library's package.json (see design-system memory `feedback_import_paths.md`).

**How to apply:** for admin CRUD work in `frontend-commerce`, load the commerce memory scope's `project_corepark.md` before designing — it has aliases, state pattern, service conventions, settings-page canonical flow, and design tokens. For design-system component APIs, load its memory scope's `project_corepark_ui_components.md`. For `frontend-survey`, no memory exists — read source directly.
