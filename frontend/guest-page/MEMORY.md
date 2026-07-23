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
- [Feature: Firebase Auth](feature_firebase_auth.md) — Backend + frontend funcionando: signIn verificado, listener de RT DB en ticket específico, initializer bloquea bootstrap hasta auth
- [Bug: Phone form gate (Jul 2026)](bug_phone_number_form_gate.md) — Backend dejó de mandar guest.phoneNumber cuando no existe; fix: usar truthy check en lugar de === null. Regla: en este proyecto los opcionales se omiten, no vienen como null
- [Feature: Stripe Integration](feature_stripe_integration.md) — 4th gateway; two flows on same branch — pay-at-checkout (PaymentIntent, done) + CoF "Unlock your valet pass" gate (SetupIntent, FE done, backend endpoint pending)
- [Boot orchestration on ticket page](project_boot_orchestration.md) — ticket.component owns forkJoin(config, get-cfg) → optional gateway fetch → finalize loader; children never fetch boot config
- [MatDialog panelClass scroll convention](project_mat_dialog_scroll.md) — Every new dialog with custom panelClass needs a matching rule in _custom-mat-dialog.scss or the panel won't scroll
