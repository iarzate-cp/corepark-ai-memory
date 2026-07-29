---
name: Feature — Stripe Integration (branch feature/stripe-integration)
description: Fourth payment gateway on the guest page. Two flows on same branch — (1) pay-at-checkout (PaymentIntent, 2026-07-14) and (2) card-on-file "Unlock your valet pass" gate (SetupIntent, 2026-07-23, frontend fully wired end-to-end against real backend endpoint).
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

Second flow on the same branch. Same product context (Stripe, Connect, Overnight tickets) but a different Stripe API: **SetupIntent** to save a `PaymentMethod` up-front, so backend can charge off-session later. Fully wired end-to-end against the real backend endpoint (2026-07-23 pm).

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
- **Setup endpoint (implemented):** `POST /payment/stripe/web/card-on-file` with `Operator-Id` + `Location-Id` headers. Body `{ ticket, consentTextVersion }`. Response envelope `{ code, data }`:
  ```json
  {
    "clientSecret": "seti_..._secret_...",
    "setupIntentId": "seti_...",
    "cardOnFileReference": "uuid",
    "publishableKey": "pk_live_...",
    "accountId": "acct_..." | null
  }
  ```
  Same Connect pattern as `get-parameters`: type 1 returns CorePark pk + operator's connected `accountId`; **type 2 returns CLIENT platform's pk and `accountId: null`** (so `loadStripe` must pass `stripeAccount` only when it's non-null). `consentTextVersion` is a required audit trail — card networks require explicit consent for off-session reuse, and the guest must have seen the corresponding copy version.
- **Self-healing POST:** re-POSTing for the same ticket cancels the previous PENDING SetupIntent and replaces the record. Useful for PENDING re-entry.
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
- `core/models/definitions/stripe.ts` — added `StripeCardDisplay`, `StripeCardOnFile`, `StripeCardOnFileResponse`, `StripeCofSetupRequest`, `StripeCofSetup`, `StripeCofSetupResponse`

### Service
- `core/services/payments.ts`:
  - `stripeCardOnFile(ticket: string)` — GET, uses `HttpRoutes.StripeCardOnFile` + `ticketInfoState.headers()` + `{ params: { ticket } }`, `.pipe(map(r => r.data))`. Also used for post-confirmSetup polling.
  - `stripeCofSetup(ticket: string, consentTextVersion: string)` — POST, same `HttpRoutes.StripeCardOnFile` entry (path is shared, only method differs), body `{ ticket, consentTextVersion }`, headers from `ticketInfoState.headers()`, returns `StripeCofSetup` (clientSecret/setupIntentId/cardOnFileReference/publishableKey/accountId).

### Setup + confirm + polling flow (implemented in `unlock-valet-pass.component.ts`)
On component `ngOnInit`:
1. `stripeCofSetup(ticket, 'cof-consent-v1')` fires.
2. On success → `loadStripe(publishableKey, accountId ? { stripeAccount: accountId } : undefined)` — nullable `accountId` MUST be handled (type 2 connection returns null).
3. `stripe.elements({ clientSecret })` + `create('payment')` + `mount(#paymentElement)`. `ready` event flips `isElementReady` → CTA enables (assuming consent already ticked).

**Consent checkbox (2026-07-28):** `hasConsent = signal(false)` gates the CTA — button stays disabled until the user ticks a checkbox rendered above it (native `<input type="checkbox">` inside a `<label>`, outside the Stripe iframe as it must be). `onSaveCard` early-returns if `!hasConsent()` as belt-and-suspenders. See "Consent gate — known limitation" below for the compliance gap this leaves open.

On "Save card" click:
4. `stripe.confirmSetup({ elements, confirmParams: { return_url: window.location.href }, redirect: 'if_required' })`.
5. On `setupIntent.status === 'succeeded'` → start polling:
   - `race(interval(1000).pipe(switchMap → GET card-on-file, filter hasActiveCardOnFile, take(1)), timer(10_000).pipe(map(() => null)))`.
   - On hit → set `StripeCardOnFileState.cardOnFile` → matrix computeds flip, gate closes reactively, ticket UI + `<stripe-saved-card>` render.
   - On 10s timeout → snackbar "Confirmation is taking longer than usual. Please refresh in a moment." (backend webhook may still land later; user refresh resolves).

Payment Element `create('payment', options)` config (2026-07-28 revision — `fields.billingDetails` removed to keep Stripe's `auto` default):
```ts
elements.create('payment', {
  layout: { type: 'tabs', defaultCollapsed: false },
  terms: { card: 'never', applePay: 'never', googlePay: 'never' }, // our own consent copy renders below
  wallets: { applePay: 'auto', googlePay: 'auto' },
})
```
Previously carried `fields: { billingDetails: { address: { country: 'never', postalCode: 'auto' } } }`. That opt-out required us to supply `country` in `confirmParams.payment_method_data.billing_details.address.country`, which the backend doesn't expose. Switching to `if_required` was suggested by Stripe support but "potentially impacts authorization rates" and can drive higher network fees on cost-plus plans. Default `auto` optimizes both and requires no override — see `project_stripe_billing_details.md`.

Constants live at the top of the component file:
```ts
const COF_CONSENT_VERSION = 'cof-consent-v1'
const POLL_INTERVAL_MS = 1000
const POLL_TIMEOUT_MS = 10000
```

### Error handling
Backend does not expose stable error codes for this endpoint (only HTTP 400/404/504 with generic bodies), so the FE shows generic snackbar messages via `#showError()`. If backend later adds a `HttpErrorStripeCofCodes` enum, wire a specific-message map inside `#initSetup` and `onSaveCard`.

**`confirmSetup` call shape (2026-07-28 revision — return_url + real error surfaced):**
```ts
this.#stripe.confirmSetup({
  elements: this.#elements,
  confirmParams: { return_url: window.location.href },
  redirect: 'if_required',
})
  .then((result) => {
    if (result.error) {
      if (result.error.type !== 'validation_error') {
        this.#showError(result.error.message ?? 'Could not save the card. Please try again.')
      }
      this.isSubmitting.set(false)
      return
    }
    // ... setupIntent.status === 'succeeded' branch
  })
  .catch((error: { message?: string }) => {
    this.#showError(error?.message ?? 'Could not save the card. Please try again.')
    this.isSubmitting.set(false)
  })
```
Rules:
- `confirmParams.return_url` **is required** — even with `redirect: 'if_required'`, Link and 3DS trigger a redirect and Stripe throws without it. Silent failure otherwise.
- The Payment Element already renders inline red errors under each field for `validation_error` (empty/invalid number, expired card, etc.) — showing a snackbar on top is redundant and noisy. Only `card_error` / `api_error` / `api_connection_error` warrant a snackbar.
- In the `.catch`, surface `error.message` from Stripe if present; the generic fallback hid the real cause during debugging. Same principle applies to any future `confirmPayment`-style call.

### Backend gotcha — Bank tab + Link form can't be disabled from FE
When the connected account has `us_bank_account` and `link` payment methods enabled, the Payment Element renders:
- A **"Banco" tab** next to "Tarjeta" (with a "USD 5" badge — the ACH minimum).
- A full **Link signup form** (email / phone / full name + "Condiciones y Política de privacidad" link) injected inside the card tab.

**Neither is disableable from FE when using `stripe.elements({ clientSecret })` mode.** `wallets` in `create('payment', ...)` only exposes `applePay` and `googlePay` — no `link` key. `paymentMethodTypes` on elements-level is deferred-intent-only.

**Fix must happen on backend when creating the SetupIntent:**
```python
stripe.SetupIntent.create(
    customer=customer_id,
    payment_method_types=["card"],  # explicit — hides Bank tab AND Link form
    usage="off_session",
    ...
)
```
Alternatively, disable Link + US bank in the connected account Dashboard settings — but per-request `payment_method_types=["card"]` is cleaner.

**Do NOT migrate to Deferred Intent flow just to solve this** — big rewrite, breaks the pattern shared with `stripe-payment.component.ts` (pay-at-checkout). Backend fix is one line.

### Components
- **`shared/components/unlock-valet-pass/`** — full-page card, fully wired against Stripe:
  - **Dual-logo header** ported from `feature/guest-vehicle-info-edit`: grid `1fr auto 1fr` (partner logo | white separator | Corepark), skeleton shimmer while S3 HEAD is in flight, `--solo` modifier when resolved without partner logo (collapses to Corepark centered). Fetches via `HeaderImageService.getImage()` in `ngOnInit`.
  - Ticket badge (`#6012`), title/subtitle, `<div #paymentElement>` slot (with shimmer skeleton until `ready` event), **consent copy** (auto-charges authorization text), teal `SAVE CARD` CTA (disabled until element ready), footer `<lock-icon /> Encrypted and processed securely`.
  - The Payment Element handles cards + wallets (Google Pay / Apple Pay / Link) automatically — the previously-static wallet button + divider + fake card inputs were **removed** in favor of the real Stripe iframe.
  - Wallet button + `credit-card-icon` were removed from `unlock-valet-pass.imports.ts` (only `lock-icon` remains).
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

## Known gaps (frontend can't close without backend or product decision)

### Consent gate — timing mismatch with guidance
Guidance is that `POST /card-on-file` (with `consentTextVersion`) should fire in the same user gesture as `confirmSetup`, so consent is only recorded when the user actively saves. Today's flow calls `stripeCofSetup(...)` in `ngOnInit` (needed to obtain `publishableKey` + `clientSecret` up front to mount the Element). The UI does gate the save behind the checkbox, but the SetupIntent + `consentTextVersion` are already created at page load. To fully align:
- Backend adds a publishable-key-only endpoint (or embed pk in env if platform-wide), or
- Two-phase COF setup: init (no consent) → confirm (with `consentTextVersion` at click).

### CoF exists but Pay dialog still asks for a card
When `hasActiveCard()` is true and the guest clicks Pay, `stripe-payment` mounts a fresh empty Payment Element via `stripeGetParameters(ticket, tipAmount)` — the Element does not know about the saved PaymentMethod, so the guest is asked to re-enter card details. Kills the CoF value proposition. Fix requires backend cooperation, options:
1. New endpoint `POST /payment/stripe/web/charge-card-on-file { ticket, tipAmount }` that confirms the PI server-side with the saved PM (`off_session: true`) and returns success / `requires_action` (3DS). Frontend skips the Element entirely.
2. Existing endpoint returns a PI already carrying `payment_method` set to the CoF, so FE calls `stripe.confirmCardPayment(clientSecret)` without Elements.
3. Backend exposes a Customer Session so `stripe.elements({ clientSecret, customerSessionClientSecret })` renders the saved card as a pre-selected option in the Element.
Blocked pending backend contract — user prompted to confirm what path exists.

### Backend `getPaymentDetail` returns malformed `tipConfig` on some locations
Observed 2026-07-28 on operator=8 / location=16 (Evolution Test, ticket=11, rate=CoF $1.5):
```json
"tipConfig": { "allowTipping": true, "amounts": [null] }
```
Missing `type` (`'P'` or `'F'`) and `customAmountAllowed`; `amounts` array contains only `null`. Frontend template shows a single empty chip labelled `%` (fallback in `@else` renders `{{ tip }}%` with `tip === null`). Fix belongs on backend — should return either a complete `tipConfig` for locations with configured tips, or `allowTipping: false` for locations without. Defensive FE filter (skip nulls + treat missing `type` as no tipping) can bridge until backend is fixed.

## Open questions (blocked by product / not yet decided)

1. **Backend restrict `payment_method_types=["card"]`** on the CoF SetupIntent — see "Backend gotcha" above. Until this lands, guests see the "Banco" tab + full Link signup form injected inside the card tab. Pending request to backend team.
2. **Consent copy version + text** — currently a placeholder string (`'By tapping Save card you authorize ... until you check out or remove the card.'`) with version constant `cof-consent-v1`. Legal/Product must supply the official copy AND bump the version string when it changes; the version travels in the POST body as audit trail.
3. **What "Pay" does when CoF is active** — product decision still open. Three options laid out: (A) hide the Pay button entirely (CoF implies backend charges off-session at checkout, no manual pay needed), (B) one-tap off-session charge via a new endpoint, (C) confirmation modal → same charge. Leaning toward (A) but not confirmed.
4. **PENDING re-entry UX** — currently the setup POST fires unconditionally on component mount, and the backend self-heals (cancels the old PENDING SetupIntent). No explicit "previous attempt didn't confirm" message shown. Silent re-offer works; product can revisit if guests report confusion.
5. **10s poll timeout behavior** — currently shows "Confirmation is taking longer than usual" snackbar and leaves the gate open. If the webhook lands afterward, the guest must refresh to see the gate close. Alternative: continue polling indefinitely in the background with a longer interval. Kept simple for now.

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
