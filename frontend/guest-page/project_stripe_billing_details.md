---
name: Stripe card-on-file — billingDetails handling
description: For Stripe Payment Element in card-on-file flow, do not override fields.billingDetails; the default (auto) preserves best authorization rates and avoids extra network fees vs if_required
type: project
---

Stripe card-on-file (SetupIntent) Payment Element should let Stripe handle billing details by default — do not set `fields.billingDetails` to `if_required` nor opt individual fields to `'never'`. `confirmSetup` needs `confirmParams.return_url` set to `window.location.href` (or similar); no `payment_method_data.billing_details.address.country` required when there's no `'never'` opt-out.

**Why:** Setting individual fields to `'never'` requires us to supply the data in `confirmParams.payment_method_data.billing_details.address.*`, forcing knowledge of the location's country the backend doesn't currently expose. `'if_required'` bypasses that but Stripe warns it "potentially impacts authorization rates" and can drive "higher network fees" on cost-plus plans. Default `auto` optimizes both.

**How to apply:** In `unlock-valet-pass.component.ts` (and any future Stripe Element in this codebase), leave the `fields.billingDetails` block off entirely. Keep `terms.card/applePay/googlePay: 'never'` for UX and `wallets: auto`. In `confirmSetup`, always pass `confirmParams: { return_url: window.location.href }` and `redirect: 'if_required'` so Link/3DS redirects work. Surface `error.message` from the `.catch` too — the generic fallback hides the real Stripe cause.
