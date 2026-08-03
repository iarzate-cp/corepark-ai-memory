# Memory Index — frontend-guest-page

> **⚠️ Memory base moved.** The authoritative memory for this project now lives in the git-tracked repo:
> **`/Users/israel/Dev/corepark-ai-memory/frontend/guest-page/`**
>
> Read from there. Write to there. Do NOT edit files under `~/.claude/projects/-Users-israel-Dev-frontend-guest-page/memory/` — this local folder is only a stub kept in sync for auto-injection. All new memories, edits, and deletions must happen against the repo path so they can be committed, pushed, and shared with the team's other Claude sessions.
>
> Index below reflects the repo's `MEMORY.md`. If they diverge, the repo wins.

- [Project Overview](project_overview.md) — Purpose, tech stack (Angular 19, Signals, 3 payment gateways), environments
- [Architecture Patterns](project_architecture.md) — Standalone, signal state, functional interceptors, initializers, theming, lazy routing
- [Payment Flows](project_payment_flows.md) — Square, Windcave (cross-tab popup), FreedomPay HPC iframe flows
- [State Classes](project_state_classes.md) — All 14 signal-based state services and their locations
- [Commit workflow](feedback_commit_workflow.md) — After a task, provide commit message text only; never run git commit
- [Prefer git switch over checkout](feedback_git_switch.md) — Use `git switch` for branch navigation, not `git checkout`
- [Memory base is corepark-ai-memory](feedback_sync_ai_memory_repo.md) — All memory reads/writes go to /Users/israel/Dev/corepark-ai-memory/frontend/guest-page/ (repo is source of truth)
- [Guest Page Flow](guest_page_flow.md) — Full end-to-end flow: routing, data load, two API shapes (ticket-info vs get-cfg), request car, payment, B&F SMS
- [Feature: Check-In + B&F SMS](feature_checkin_bf_sms.md) — Branch feature/check-in: Guest Profile hides minutes UI, B&F triggered on backend; open item on post-request state
- [Feature: allow-requests branch](feature_allow_requests.md) — get-cfg integration, GuestCfgState, allowMinutesRequest/allowDateTimeRequest, partnerProcessPayments, leavingIn source, pay-button bug fix
- [UUID Redirect — Mistyped Printed UUID](project_uuid_redirects.md) — 7f64fe6 → 07f64fe6: missing leading zero on printed material, fixed in uuidSetter()
- [UUID Redirect Flow — Origin & Backend Migration](project_uuid_redirect_flow.md) — Why uuidSetter() exists (multi-tenant printed QRs, UUID migration under live QR, misprints) and why it moves to ms-valet-service resolver + ms-backoffice-service CRUD; includes proposed `company.location_redirect` DDL modeled on `DDL_RELEASE_171_EV_CHARGING_MODULE.sql`
- [Feature: Firebase Auth](feature_firebase_auth.md) — Backend + frontend funcionando: signIn verificado, listener de RT DB en ticket específico, initializer bloquea bootstrap hasta auth
- [Bug: Phone form gate (Jul 2026)](bug_phone_number_form_gate.md) — Backend dejó de mandar guest.phoneNumber cuando no existe; fix: usar truthy check en lugar de === null. Regla: en este proyecto los opcionales se omiten, no vienen como null
- [Feature: Stripe Integration](feature_stripe_integration.md) — 4th gateway; two flows on same branch — pay-at-checkout (PaymentIntent) + CoF "Unlock your valet pass" gate (SetupIntent); both wired end-to-end. Backend must set payment_method_types=["card"] to hide Bank tab + Link form
- [Boot orchestration on ticket page](project_boot_orchestration.md) — ticket.component owns forkJoin(config, get-cfg) → optional gateway fetch → finalize loader; children never fetch boot config
- [MatDialog panelClass scroll convention](project_mat_dialog_scroll.md) — Every new dialog with custom panelClass needs a matching rule in _custom-mat-dialog.scss or the panel won't scroll
- [Scalar MCP has stale Payment Service spec](reference_scalar_mcp_payment_spec.md) — Stripe endpoints missing from the MCP workspace; use api-specs.corepark.com web UI / PDF export instead
- [Stripe billingDetails handling](project_stripe_billing_details.md) — Do not override fields.billingDetails on Payment Element; default `auto` gives best auth rate. `if_required` hurts approvals; `'never'` needs backend country
- [Missing @types/crypto-js on develop — RESOLVED](project_missing_crypto_js_types.md) — Historical; fixed 2026-07-28 in chore commit 61bfa0f1. Kept for regression reference
- [ticket.compensation hides Pay button](project_compensation_flag_hides_pay_button.md) — Backend stamps compensation=true when charge comes from elsewhere (CoF, comps, validations); existing displayPayButton rule already respects it — do NOT add a CoF-specific check
- [Feature: Hotel Info gate (branch feature/ticket-info-hotel-room)](feature_hotel_info_gate.md) — Room + hotel expected-checkout required before Request Car on hotel locations. Backend: 2 commits in ms-valet-service (ticket-info exposes fields + request-car persists them). Frontend: new `<hotel-info-form>`, `hasHotelInfo` gate in RequestCarState. Ships silently disabled until backend exposes `requireHotelInfo` flag on GuestConfig
