---
name: Feature — Stripe Integration (branch feature/stripe-integration)
description: Fourth payment gateway on the guest page. Two flows on same branch — (1) pay-at-checkout (PaymentIntent, 2026-07-14) and (2) card-on-file "Unlock your valet pass" gate (SetupIntent, 2026-07-23, frontend done, backend pending).
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

---

# Card-on-file "Unlock your valet pass" (2026-07-23)

Second flow on the same branch. Same product context (Stripe, Connect, Overnight tickets) but a different Stripe API: **SetupIntent** to save a `PaymentMethod` up-front, so backend can charge off-session later. Frontend is fully built; **backend endpoint that creates the SetupIntent is still pending** — the Payment Element / confirmSetup wiring is the last mile.

## Contract with backend

- **Read endpoint (implemented):** `GET /payment/stripe/web/card-on-file?ticket={ticket}` with `Operator-Id` + `Location-Id` headers. Response envelope `{ code, data }`, data shape:
  ```json
  {
    "eligible": true,
    "ineligibleReason": null,
    "hasActiveCardOnFile": true,
    "cardOnFileStatus": "ACTIVE" | "PENDING" | null,
    "card": { "brand": "visa", "last4": "4242", "expMonth": 7, "expYear": 2027 } | null
  }
  ```
  Cache-Control: `no-store`. **Never errors** for ineligible tickets — just returns `eligible: false` with a reason.
- **Setup endpoint (still pending on backend):** must accept `{ ticket }` and return `{ clientSecret, publishableKey, accountId }` (same Connect tuple as `get-parameters`) so the frontend can mount a Payment Element in `mode: 'setup'` and call `stripe.confirmSetup()`. Once Apple/Google Pay wallet enablement is finished on connected accounts, the Payment Element renders wallets automatically.
- **Eligibility rules (backend enforced):** ticket OPEN + Overnight rate + valet-collected (not partner/PMS) + location connected to Stripe. Reasons enum: `TICKET_NOT_FOUND | NOT_OVERNIGHT_RATE | PARTNER_COLLECTED | GATEWAY_NOT_CONFIGURED`.
- **PENDING reconciliation:** every GET reconciles against Stripe. After a `confirmSetup` we re-consult the GET; the read authoritatively flips `hasActiveCardOnFile` (no webhook dependency for UI). On re-entry with `PENDING`, we offer the form again — new POST replaces the stale record.

## Decision matrix (drives the UI)

Computed in `StripeCardOnFileState`:

| response                                          | signal                          | ticket page renders                              |
| ------------------------------------------------- | ------------------------------- | ------------------------------------------------ |
| `eligible && !hasActiveCardOnFile`                | `showUnlockGate()` = true       | `<unlock-valet-pass>` **only** — hide app-header, app-footer, rest of ticket UI. Dark bg #212121, host `display: flex; align-items: center`. |
| `eligible && hasActiveCardOnFile`                 | `showSavedCard()` = true        | Normal ticket page + `<stripe-saved-card>` chip between request-car and ticket-actions. |
| `!eligible` (any reason)                          | both = false                    | Normal ticket page, no CoF UI.                   |

## Frontend architecture

### State
- **`core/states/stripe-card-on-file-state.ts`** — `StripeCardOnFileState`:
  - Primary: `cardOnFile = signal<StripeCardOnFile | null>(null)`
  - Atomic computeds: `isEligible`, `hasActiveCard`, `status`, `ineligibleReason`, `card`
  - Decision-matrix computeds: `showUnlockGate`, `showSavedCard`, `isPending`
  - `reset()` clears everything

### Types & enums
- `core/models/enums/stripe.ts` — added `CardOnFileStatus` (`Active`, `Pending`) and `StripeIneligibleReason` (four values above)
- `core/models/definitions/stripe.ts` — added `StripeCardDisplay`, `StripeCardOnFile`, `StripeCardOnFileResponse`

### Service
- `core/services/payments.ts` — `stripeCardOnFile(ticket: string)` uses `HttpRoutes.StripeCardOnFile` + `ticketInfoState.headers()` + `{ params: { ticket } }`, `.pipe(map(r => r.data))`

### Components
- **`shared/components/unlock-valet-pass/`** — full-page card:
  - **Dual-logo header** ported from `feature/guest-vehicle-info-edit`: grid `1fr auto 1fr` (partner logo | white separator | Corepark), skeleton shimmer while S3 HEAD is in flight, `--solo` modifier when resolved without partner logo (collapses to Corepark centered). Fetches via `HeaderImageService.getImage()` in `ngOnInit`.
  - Ticket ticket-badge (`#6012`), title/subtitle, black **"Pay with G Pay"** button (colored Google G inline SVG), `or enter manually` divider, static Card number / MM/YY / CVV inputs (with CVV `?` help), teal `ADD CARD` CTA (disabled), footer `<lock-icon /> Encrypted and processed securely`.
  - The static form is a **visual placeholder** for the Stripe Payment Element that will mount in `mode: 'setup'` once the setup endpoint exists. Same `<div #paymentElement>` slot pattern as `stripe-payment.component.ts`.
- **`shared/components/stripe-saved-card/`** — chip for `showSavedCard()`:
  - Panel with left brand-color accent bar (Visa blue / Mastercard orange / Amex blue / Discover orange via `[attr.data-brand]` + `color-mix` for badge tint)
  - Caption "PAYMENT METHOD ON FILE" with lock icon, brand name (mapped from slug), •••• last4 (large bold), right-aligned `EXPIRES MM/YY` (tabular-nums)
  - Reads directly from `StripeCardOnFileState`
- **`shared/icons/lock-icon.ts`** — new outline lock icon

### Ticket page integration (`pages/ticket/`)
- `ticket.component.ts` hosts the boot orchestrator (see next section).
- Host binding: `host: { '[class.unlock-gate]': 'showUnlockGate()' }`.
- `ticket.component.scss` — `:host.unlock-gate { min-height: 100dvh; background: var(--color-grey-500); display: flex; align-items: center; main { width: 100% } }`. `--color-grey-500` (`#212121`) is the closest existing token to the desired `#202020`.
- `ticket.component.html` — when `showUnlockGate()` is true, hide `<app-header />` and `<app-footer />` (T&C) and render `<unlock-valet-pass @SoftRise />`. Otherwise the normal ticket layout renders and `<stripe-saved-card />` slots in between request-car and ticket-actions if `showSavedCard()`.

## Boot orchestration (2026-07-23 refactor — anti-flicker)

**Before:** `ticket-actions.ngOnInit` fetched `getGuestPageConfiguration`; ticket page fetched `getGuestPageCfg` in parallel; loader hid immediately after `getTicketInfo`. Result: the page briefly rendered "normal" content, then flipped to unlock gate → visible flicker.

**After:** `ticket.component` owns the entire boot chain. `ticket-actions` is now purely presentation (no `OnInit`, no `LocationDetailService`/`PaymentsService`/`StripeCardOnFileState` injections).

```ts
#bootstrapConfigs() {
  forkJoin({
    config: this.#locationDetailService.getGuestPageConfiguration(),
    guestCfg: this.#locationDetailService.getGuestPageCfg(),
  }).pipe(
    switchMap(({ config, guestCfg }) => {
      setLocalStorage(STORAGE_KEYS.CONFIGURATION, config)
      this.#ticketActionsState.configuration.set(config)
      this.#guestCfgState.guestConfig.set(guestCfg)
      if (config?.paymentGateway !== PaymentGateway.Stripe) return of(null)
      const ticket = this.#ticketInfoState.ticketInfo()?.ticket?.ticket
      if (!ticket) return of(null)
      return this.#paymentsService.stripeCardOnFile(ticket)
    }),
    finalize(() => this.#loaderState.hide()),
    takeUntilDestroyed(this.#destroyRef)
  ).subscribe({
    next: (cardOnFile) => { if (cardOnFile) this.#stripeCardOnFileState.cardOnFile.set(cardOnFile) },
    error: (err) => this.#loggerService.error('Failed to bootstrap configs', err),
  })
}
```

- Called from `getTicketInfo` success handler **after** `#updateTicketInfo(ticketInfo)` (so headers + ticket string are ready).
- `finalize` fires on both success **and** error → loader always hides, never gets stuck.
- **Loader is now solid `var(--color-white)`**, no transparency/blur — covers all boot activity opaquely.
- The old `loaderState.hide()` in `getTicketInfo.next` and the old `#fetchGuestPageCfg` method were removed.

## Transition animations (`shared/animations/fades.ts`)

Three triggers now:

```ts
fadeInOut  // existing — used on <main @FadeInOut>
fadeOut    // new — used on <loader @FadeOut />           240ms ease-out on :leave only
softRise   // new — used on <unlock-valet-pass @SoftRise /> 280ms cubic-bezier(0.16, 1, 0.3, 1) on :enter — fade + 8px translateY
```

Timing at the boot completion moment:
- `t=0`: `finalize` fires → `loaderState.hide()` AND `showUnlockGate()` flips true in the same tick.
- `t=0-240ms`: loader dissolves (`@FadeOut`).
- `t=0-280ms`: unlock rises from behind (`@SoftRise`).
- Overlap eliminates any cut; user perceives the loader "melting into" the unlock screen.

## Files added/modified today (2026-07-23)

**Added:**
- `core/states/stripe-card-on-file-state.ts`
- `shared/components/unlock-valet-pass/{index.ts, unlock-valet-pass.component.ts,html,scss, unlock-valet-pass.imports.ts}`
- `shared/components/stripe-saved-card/{index.ts, stripe-saved-card.component.ts,html,scss, stripe-saved-card.imports.ts}`
- `shared/icons/lock-icon.ts`

**Modified:**
- `core/models/enums/stripe.ts` (+`CardOnFileStatus`, +`StripeIneligibleReason`)
- `core/models/enums/http-routes.ts` (+`StripeCardOnFile`)
- `core/models/definitions/stripe.ts` (+`StripeCardDisplay`, +`StripeCardOnFile`, +`StripeCardOnFileResponse`)
- `core/services/payments.ts` (+`stripeCardOnFile()`)
- `pages/ticket/ticket.component.ts` (bootstrap orchestrator, host binding, `@SoftRise`)
- `pages/ticket/ticket.component.html` (unlock gate branching, hide header/footer when gated)
- `pages/ticket/ticket.component.scss` (`:host.unlock-gate` styles)
- `pages/ticket/ticket.imports.ts` (+`UnlockValetPass`, +`StripeSavedCard`)
- `shared/components/ticket-actions/ticket-actions.component.ts` (removed ngOnInit fetch, purged 3 injections)
- `shared/components/loader/loader.component.scss` (solid white bg, dropped `backdrop-filter`)
- `app.component.ts` (+`@FadeOut` on loader element)
- `shared/animations/fades.ts` (+`fadeOut`, +`softRise`)

## Open questions (blocked by backend / product)

1. **SetupIntent endpoint** — need `POST /payment/stripe/web/card-on-file` (or similar) that returns `{ clientSecret, publishableKey, accountId }`. Without it the `Add card` button is a no-op and the static form is decorative. Once available: mirror `stripe-payment.component.ts` but with `stripe.elements({ mode: 'setup', clientSecret })` + `stripe.confirmSetup()` + polling the read endpoint until `hasActiveCardOnFile` flips.
2. **What "Pay" does when CoF is active** — product decision still open. Three options laid out: (A) hide the Pay button entirely (CoF implies backend charges off-session at checkout, no manual pay needed), (B) one-tap off-session charge via a new endpoint, (C) confirmation modal → same charge. Leaning toward (A) but not confirmed.
3. **PENDING re-entry UX** — silent re-offer of the form vs. explicit "your previous attempt didn't confirm, try again" message. Both would work; not decided.

## Discarded alternatives (do not resurrect without reason)

- **Dedicated `/unlock/:locationUUID/:ticket` route with `UnlockLayout`** mirroring `validation-layout`: built and then reverted. User preferred toggling visibility within the ticket page — "de esa manera, tenemos mejor manera manejar las cosas para futuras integraciones". Files deleted: `pages/unlock/`, `shared/layouts/unlock-layout/`, and the route. If you find yourself building this again, revisit the decision explicitly with the user.

## Open items / gotchas (pay-at-checkout flow, still relevant)

1. **Config route prefix mismatch (unresolved):** frontend uses `/backoffice/payment/guest-page/configuration` (with prefix) and it works on dev. Backend docs shared show `/payment/guest-page/configuration` (without `/backoffice`). Might be gateway/proxy rewriting — if backend later renames, update `HttpRoutes.GuestPageConfiguration`.
2. **No optimistic refresh** (see UX section) — post-payment ticket state can be stale until manual reload.
3. **Apple/Google Pay wallets** require: (a) domain registered with Stripe **per connected account**, (b) enabled on connected account's Dashboard, (c) backend PaymentIntent uses `automatic_payment_methods: { enabled: true }` (not `payment_method_types: ['card']`).
4. **Payment method types are backend-controlled.** Frontend Payment Element shows what the PaymentIntent allows (currently: card + Link + direct debit + Klarna). To restrict to card only, backend adjusts the PaymentIntent creation.
5. **Test cards:** `4242 4242 4242 4242` (success), `4000 0025 0000 3155` (3DS challenge), `4000 0000 0000 9995` (declined). Only work with `pk_test_...` publishable key — check the prefix in `get-parameters` response.

## References
- Scalar API docs MCP registered locally 2026-07-23: `scalar` HTTP transport → `https://api.scalar.com/vector/mcp/0d0afc1e-6280-47ea-a99f-b5b2ea9ead62`. Restart Claude Code to pick it up. Use for looking up exact request/response shapes for pending endpoints.
- Header dual-logo pattern originates in branch `feature/guest-vehicle-info-edit` under `shared/layouts/vehicle-info-layout/`. Same skeleton + solo pattern replicated here.
