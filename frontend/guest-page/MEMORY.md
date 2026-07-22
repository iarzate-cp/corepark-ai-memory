# Memory Index — frontend-guest-page

- [Project Overview](project_overview.md) — Purpose, tech stack (Angular 19, Signals, 3 payment gateways), environments
- [Architecture Patterns](project_architecture.md) — Standalone, signal state, functional interceptors, initializers, theming, lazy routing
- [Payment Flows](project_payment_flows.md) — Square, Windcave (cross-tab popup), FreedomPay HPC iframe flows
- [State Classes](project_state_classes.md) — All 14 signal-based state services and their locations
- [Commit workflow](feedback_commit_workflow.md) — After a task, provide commit message text only; never run git commit
- [Guest Page Flow](guest_page_flow.md) — Full end-to-end flow: routing, data load, two API shapes (ticket-info vs get-cfg), request car, payment, B&F SMS
- [Feature: Check-In + B&F SMS](feature_checkin_bf_sms.md) — Branch feature/check-in: Guest Profile hides minutes UI, B&F triggered on backend; open item on post-request state
- [Feature: allow-requests branch](feature_allow_requests.md) — get-cfg integration, GuestCfgState, allowMinutesRequest/allowDateTimeRequest, partnerProcessPayments, leavingIn source, pay-button bug fix
- [UUID Redirect — Mistyped Printed UUID](project_uuid_redirects.md) — 7f64fe6 → 07f64fe6: missing leading zero on printed material, fixed in uuidSetter()
- [Feature: Firebase Auth](feature_firebase_auth.md) — Backend + frontend funcionando: signIn verificado, listener de RT DB en ticket específico, initializer bloquea bootstrap hasta auth
- [Bug: Phone form gate (Jul 2026)](bug_phone_number_form_gate.md) — Backend dejó de mandar guest.phoneNumber cuando no existe; fix: usar truthy check en lugar de === null. Regla: en este proyecto los opcionales se omiten, no vienen como null
- [Feature: Stripe Integration](feature_stripe_integration.md) — 4th gateway; Payment Element + auto-capture PaymentIntent; backend closes ledger via webhook; mirrors Freedompay pattern; open items on route prefix and wallets
- [MatDialog panelClass scroll convention](project_mat_dialog_scroll.md) — Every new dialog with custom panelClass needs a matching rule in _custom-mat-dialog.scss or the panel won't scroll
