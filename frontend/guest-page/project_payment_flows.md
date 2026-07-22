---
name: Payment Flows — frontend-guest-page
description: How Square, Windcave, and FreedomPay payment flows work in the app
type: project
originSessionId: 5af14dee-5704-4786-bda3-2b912b2b7349
---
Payment gateway is determined by backend config in `GuestPageConfigState`. Three supported gateways:

**Square:** `PaymentComponent` — Square Web SDK tokenizes card → POST `/partner/guest/payment/pay-ticket` → `PaidTicketDialog`.

**Windcave:** Creates HPP session via POST `/payment/windcave/hpp/session` → opens popup window → popup writes result to `localStorage['WINDCAVE_STATUS']` → main window reads via `StorageEvent` (cross-tab communication) → `WindcaveStatusDialog`.

**FreedomPay:** POST `/payment/freedompay/hpc/init` → receive sessions array → render iframes (Card/GooglePay/ApplePay) → `FreedompayState` manages selected method + tipping → POST `/payment/freedompay/hpc/submit`.

**Overnight rate blocking:** Tickets with `rate.id === RateType.Overnight` (6) cannot be paid through the portal. Three layers of defense:
1. `displayPayButton` computed in `ticket-actions.component.ts` returns false — button is hidden.
2. `PayButtonComponent.onClick()` opens `OvernightRateDialog` if button is somehow visible — tells guest to pay at cashier.
3. `paymentGuard` in `core/guards/payment.ts` redirects to `/ticket/:uuid/:ticket` if user navigates directly to `/payment`.

**How to apply:** When working on payment features, identify the active gateway first from `GuestPageConfigState`. Each gateway has its own state (`FreedompayState`), dialog components, and service methods in `PaymentsService`. Never assume all tickets can pay — check `RateType` and `config.allowPayment` first.
