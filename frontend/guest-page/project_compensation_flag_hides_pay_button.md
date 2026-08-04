---
name: Backend uses ticket.compensation to hide the guest pay button
description: For CoF-active tickets, backend sets ticket.compensation=true on ticket-info so the existing displayPayButton rule hides the Pay CTA — no frontend CoF-awareness needed
type: project
---

Backend flags tickets as `ticket.compensation: true` in the ticket-info response whenever the amount is going to be captured by an alternate mechanism (Stripe card-on-file auto-charge at business-day cut, comps, validations, etc.). The frontend already respects this via a preexisting rule in `displayPayButton` (`shared/components/ticket-actions/ticket-actions.component.ts`):

```ts
if (ticketPrice === 0 || ticketInfo?.ticket?.compensation) return false
```

So a Stripe CoF ticket in the "already unlocked" state (`hasActiveCardOnFile: true`) naturally hides the Pay button — even when `totalToPay > 0` — because backend stamps `compensation: true` on it.

**Why:** backend owns the "who charges this" decision. `compensation: true` is the generic signal that means "don't offer a manual pay CTA; the charge is coming from elsewhere". This generalizes beyond CoF (subscriptions, comps, validations, etc.).

**How to apply:** do NOT add a `stripeCardOnFileState.hasActiveCard()` check to `displayPayButton` — it would be redundant with the compensation signal and would only cover the Stripe case, missing other compensation scenarios. If a Pay button ever needs to be *forced* visible for a CoF ticket, the fix belongs in the backend contract, not in the frontend rule.

Verified 2026-07-30 with ticket #21 on dev (rate `CoF $1.5`, location Evolution Test): payload had `totalToPay: 1.5`, `compensation: true`, `partnerProcessPayments: false`, `allowPayment: true` — Pay button correctly hidden.
