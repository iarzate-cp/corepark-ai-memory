---
name: Feature — Stripe Integration (branch feature/stripe-integration)
description: Fourth payment gateway on the guest page. Mirrors FreedomPay pattern. Payment Element + PaymentIntent + backend ledger via webhook. Optimistic UX, no confirm endpoint on our backend.
type: project
originSessionId: 867d223b-0e43-40dd-afc5-3cd8b73f7e0d
---
Branch: `feature/stripe-integration`. Fourth payment gateway added to `PaymentGateway` enum (after Square / Windcave / Freedompay). Implemented 2026-07-14.

**Why the pattern:** user explicitly asked "lo más parecido a Freedompay/Square". Every Stripe artifact mirrors the equivalent Freedompay one (state, dialog, payment component, init-error, tipping, payment-detail). This is intentional — do NOT invent a different structure for future changes here.

**How to apply:** When touching Stripe, first look at the Freedompay equivalent — behavior should stay parallel. When adding a new payment gateway, mirror the same tree.

## Contract with backend

- **Config discriminator:** `GET /backoffice/payment/guest-page/configuration` returns `{ configuration: { paymentGateway: 'STRIPE', transactionFee } }`. **No credentials here** — for Stripe this endpoint returns only the discriminator + fee. Credentials come at pay time.
- **Init at pay time:** `POST /payment/stripe/web/get-parameters` with body `{ ticket, tipAmount }`. Returns `{ data: { clientSecret, paymentIntentId, transactionReference, publishableKey, accountId, totalAmount, currency } }`.
- **No confirm endpoint on our backend** — frontend calls `stripe.confirmPayment({ elements, redirect: 'if_required' })` directly against Stripe. Backend closes the ledger via `payment_intent.succeeded` webhook.
- **Tip is chosen before init** — `tipAmount` goes in the init payload, is already included in the PaymentIntent's `totalAmount`. No post-confirm capture step (auto-capture PaymentIntent).
- **Self-healing on backend:** previous open web attempt for same ticket is cancelled and restarted. If already charged → `ERR-STRIPE-011-TICKET-ALREADY-PAID` in init response.
- **Stripe Connect:** each location has a connected account. Pass `accountId` as `stripeAccount` to `loadStripe(publishableKey, { stripeAccount: accountId })`. Apple Pay domain verification is per-connected-account.

## UX decisions

- **Optimistic post-payment:** on `confirmPayment` success → snackbar + close dialog. **No re-fetch of ticket-info.** Guest may see stale state until manual reload. User confirmed this was acceptable for now. If revisiting: add refetch of `getTicketInfo` after success (be aware webhook may take 1-2s to close ledger on backend — race condition).
- **Payment Element**, not Card Element or Checkout redirect. Renders in iframe, handles PCI on Stripe's side. Auto-detects wallets (Apple/Google Pay/Link/Klarna/direct debit) based on Dashboard config of the connected account.
- **Skeleton while iframe loads:** dialog-loader hides as soon as `get-parameters` returns; a CSS skeleton (`stripe-payment-skeleton`) covers the iframe area with shimmer animation until Stripe fires the `ready` event on the Payment Element.

## Files created

- `core/models/enums/stripe.ts` — `StripeInitError`
- `core/models/enums/http-error-services/stripe.ts` — `HttpErrorStripeCodes`
- `core/models/definitions/stripe.ts` — `InitStripeWebPaymentRequest`, `StripeGetParameters`, `StripeGetParametersResponse`
- `core/states/stripe-state.ts` — signals + computed for tipping/breakdown (mirror of `FreedompayState`, minus iframe-specific fields)
- `shared/components/stripe-tipping/` — mirror of `freedompay-tipping`
- `shared/components/stripe-payment-detail/` — mirror of `freedompay-payment-detail`
- `shared/components/stripe-payment/` — mounts Payment Element via `@stripe/stripe-js`
- `shared/components/stripe-init-error/` — mirror of `freedompay-init-error`
- `shared/components/stripe-dialog/` — container: tipping → payment (mirror of `freedompay-dialog`)

**Modified:** `payment-gateway.ts` (+`Stripe`), `http-routes.ts` (+`StripeGetParameters`), `payments.ts` (+`stripeGetParameters`), `pay-button.component.ts` (+`#openStripeDialog` branch), `package.json` (+`@stripe/stripe-js@^5.4.0`), `_custom-mat-dialog.scss` (+`.stripe-dialog { overflow-y: auto }`), `assets/images/stripe.svg` (fixed HDS CSS var → `#635BFF`).

## Open items / gotchas (as of 2026-07-14)

1. **Config route prefix mismatch (unresolved):** frontend uses `/backoffice/payment/guest-page/configuration` (with prefix) and it works on dev. Backend docs shared show `/payment/guest-page/configuration` (without `/backoffice`). Might be gateway/proxy rewriting — if backend later renames, update `HttpRoutes.GuestPageConfiguration`.
2. **No optimistic refresh** (see UX section) — post-payment ticket state can be stale until manual reload.
3. **Apple/Google Pay wallets** require: (a) domain registered with Stripe **per connected account**, (b) enabled on connected account's Dashboard, (c) backend PaymentIntent uses `automatic_payment_methods: { enabled: true }` (not `payment_method_types: ['card']`).
4. **Payment method types are backend-controlled.** Frontend Payment Element shows what the PaymentIntent allows (currently: card + Link + direct debit + Klarna). To restrict to card only, backend adjusts the PaymentIntent creation.
5. **Test cards:** `4242 4242 4242 4242` (success), `4000 0025 0000 3155` (3DS challenge), `4000 0000 0000 9995` (declined). Only work with `pk_test_...` publishable key — check the prefix in `get-parameters` response.
