---
name: Feature — allow-requests branch
description: All decisions and changes made on branch feature/allow-requests. Merged into main 2026-05-29.
type: project
originSessionId: 6e04c6e0-0e5f-498f-b21a-ddccfc286ab0
---
## Branch: `feature/allow-requests`

### What was built

#### 1. `/backoffice/guest-page/get-cfg` integration
- `LocationDetailService.getGuestPageCfg()` → POST with `{ operatorCompanyId, parkingLocationId }`
- Response stored in `GuestCfgState.guestConfig` (signal, separate from ticketInfo)
- Called from `TicketComponent.#fetchGuestPageCfg()` after ticket-info loads
- `GuestPageCfg` interface in `core/models/definitions/guest-page-config.ts`

#### 2. `allowMinutesRequest` / `allowDateTimeRequest` config flags
- Source: `GuestCfgState.guestConfig` (from get-cfg), NOT ticketInfo.config
- `allowMinutesRequest` → shows Minutes tab in request-car UI
- `allowDateTimeRequest` → shows SpecificDate (datetime-local) tab
- Both flags read via `requestOptions` computed in `RequestCarComponent` and `RequestCarDialogComponent`
- `displayRequestCarForm` in `TicketComponent` also gates on these flags

#### 3. `leavingIn` field — minutes dropdown source
- Field name in ticket-info API is `leavingIn` (not `leavingInCfg`)
- `GuestConfig.leavingIn: number[]` in `core/models/definitions/common.ts`
- Minutes dropdown always reads from `ticketInfo().config.leavingIn`
- get-cfg also returns `leavingInCfg` but it is NOT used for the dropdown

#### 4. `partnerProcessPayments` flag on `Rate`
- `Rate.partnerProcessPayments: boolean` in `core/models/definitions/common.ts`
- Controls whether Overnight rate blocks payment
- `false` = valet-collected → payment allowed
- `true` = partner collects → payment blocked
- Must be checked in all three places:
  - `paymentGuard` (`core/guards/payment.ts`)
  - `displayPayButton` computed (`ticket-actions.component.ts`)
  - `onClick()` in `pay-button.component.ts` ← bug fix: was missing this check, caused OvernightRateDialog to show for valet-collected tickets

#### 5. `GuestConfig` vs `GuestPageCfg` separation
- `GuestConfig` (ticket-info shape): `allowRequest`, `allowPayment`, `allowBackAndForth`, `leavingIn`
- `GuestPageCfg` (get-cfg shape): `allowRequest`, `allowPayment`, `allowMinutesRequest`, `allowDateTimeRequest`, `leavingInCfg`
- These are separate interfaces — do NOT merge or confuse them

### Components updated to read from `GuestCfgState`
All `allow*` flags (except `leavingIn`) now read from `GuestCfgState.guestConfig()`:
- `TicketComponent` — `displayRequestCarForm`
- `RequestCarComponent` — `allowRequest`, `requestOptions`
- `RequestCarDialogComponent` — `requestOptions`
- `TicketActionsComponent` — `displayPayButton` (allowPayment), `displayRequestCarBtn` (allowRequest), `displayTipBtn` (allowPayment)
- `PaymentComponent` — post-payment `allowRequest` check

### Bug fixed
**`OvernightRateDialog` showing for valet-collected Overnight tickets**
- Root cause: `pay-button.component.ts onClick()` only checked `rate.id === RateType.Overnight`, ignored `partnerProcessPayments`
- Fix: added `&& rate.partnerProcessPayments` to the condition
