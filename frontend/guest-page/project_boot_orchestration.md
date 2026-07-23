---
name: Ticket-page boot orchestration (anti-flicker pattern)
description: The ticket page owns the boot chain (getTicketInfo → forkJoin(config, get-cfg) → optional gateway-specific fetch → finalize loader). Loader stays visible until all bootstrap responses land. Child components never fetch boot config.
type: project
originSessionId: 9b60e26d-792d-4aa4-9aab-212bea254a10
---
# Ticket-page boot orchestration

Established 2026-07-23 during the Stripe CoF work. Replaced a legacy pattern where `ticket-actions.ngOnInit` fetched `getGuestPageConfiguration` (a page-scope concern being done by a leaf component), and the loader hid immediately after `getTicketInfo` — causing a visible flicker as gateway-specific state resolved.

## Rule

The ticket page (`pages/ticket/ticket.component.ts`) is the only place that fetches bootstrap-level data. Child components (`ticket-actions`, `unlock-valet-pass`, etc.) may fire component-scope requests in their own `ngOnInit`, but nothing that determines whether the page-level layout is different (gateway, gate visibility, cfg flags).

## Shape

Inside `getTicketInfo` success → after `#updateTicketInfo(ticketInfo)`, call `#bootstrapConfigs()`:

```ts
#bootstrapConfigs() {
  forkJoin({
    config: this.#locationDetailService.getGuestPageConfiguration(),
    guestCfg: this.#locationDetailService.getGuestPageCfg(),
  }).pipe(
    switchMap(({ config, guestCfg }) => {
      // Sync side effects: cache to storage, hydrate states
      setLocalStorage(STORAGE_KEYS.CONFIGURATION, config)
      this.#ticketActionsState.configuration.set(config)
      this.#guestCfgState.guestConfig.set(guestCfg)

      // Gateway-specific tail request. Extend here per gateway.
      if (config?.paymentGateway !== PaymentGateway.Stripe) return of(null)
      const ticket = this.#ticketInfoState.ticketInfo()?.ticket?.ticket
      if (!ticket) return of(null)
      return this.#paymentsService.stripeCardOnFile(ticket)
    }),
    finalize(() => this.#loaderState.hide()),
    takeUntilDestroyed(this.#destroyRef)
  ).subscribe({
    next: (tailResponse) => { /* set gateway-specific state */ },
    error: (err) => this.#loggerService.error('Failed to bootstrap configs', err),
  })
}
```

## Why

- **No flicker:** loader covers all boot activity opaquely; template only paints once `showUnlockGate()` / `showSavedCard()` / gateway are known.
- **Resilient to errors:** `finalize` fires on both `complete` and `error`, so the loader never hangs.
- **Clean separation:** `ticket-actions` is now presentational only — pure buttons + computeds off state. Easier to reason about, easier to test.
- **Extensible per gateway:** each new gateway just adds a branch inside `switchMap` (return `of(null)` to no-op, or return the gateway-specific fetch).

## When adding another gateway that needs a boot-time read

1. Add the branch inside the `switchMap` mapper — return the observable that resolves the gateway's bootstrap state.
2. Set the corresponding state signal in the `subscribe.next`.
3. Do NOT fetch this in a child component's `ngOnInit`; that reintroduces the flicker.
4. If the gateway's boot request can be omitted for some tickets, guard with `if (…) return of(null)`.

## Related animations (`shared/animations/fades.ts`)

- Loader has `@FadeOut` (240ms opacity :leave) → soft dissolve when `finalize` hides it.
- Unlock gate has `@SoftRise` (280ms opacity + 8px translateY :enter) → emerges from behind the fading loader.
- Overlap of the two makes the transition feel continuous instead of a hard cut.

## Anti-patterns to avoid

- **Do not** call `loaderState.hide()` inside `getTicketInfo.next` before the config chain completes — that's the flicker source.
- **Do not** put page-level config fetches in child components' `ngOnInit`.
- **Do not** replace `finalize` with `subscribe.complete` — errors would skip complete and leave the loader stuck.
