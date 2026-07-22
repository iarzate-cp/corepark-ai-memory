---
name: Guest Page — Full Flow Documentation
description: Complete end-to-end flow of the guest page: routing, data loading, components, request car, payment, B&F SMS. Updated 2026-05-29.
type: project
originSessionId: 6e04c6e0-0e5f-498f-b21a-ddccfc286ab0
---
## Entry point

URL pattern: `/ticket/:locationUUID/:ticket`
Route loads `TicketComponent` (lazy via `loadComponent`).
Two initializers already run before bootstrap: `urlParamsInitializer` and `ticketInfoInitializer` — so `TicketInfoState.ticketInfo` may already be populated before `ngOnInit`.

---

## Data loading (`TicketComponent.ngOnInit`)

1. Calls `PartnerService.getTicketInfo(uuid, ticket)` → `GET /partner/guest/ticket-info/:uuid/:ticket`
2. On success:
   - If code includes `CAR-DELIVERED` or `DATA-NOT-FOUND` → navigate to `/common/delivered`
   - Otherwise: sets `TicketInfoState.ticketInfo`, `TicketActionsState.isCarRequested`, `TicketActionsState.expectedCheckout`
   - Applies Curbstand theme if company UUID matches
3. Then calls `#fetchGuestPageCfg()` → `POST /backoffice/guest-page/get-cfg`
   - Response stored in `GuestCfgState.guestConfig` (separate from ticketInfo)
4. On error → navigate to `/common/error`

---

## Two API shapes — IMPORTANT

### ticket-info `config` → `GuestConfig` (`core/models/definitions/common.ts`)
```ts
interface GuestConfig {
  allowRequest: boolean
  allowPayment: boolean
  allowBackAndForth: boolean
  leavingIn: number[]        // minutes options for the dropdown (e.g. [1,2,3,4,5])
}
```

### get-cfg `guestPageCfg` → `GuestPageCfg` (`core/models/definitions/guest-page-config.ts`)
```ts
interface GuestPageCfg {
  allowRequest: boolean
  allowPayment: boolean
  allowMinutesRequest: boolean      // controls Minutes tab visibility
  allowDateTimeRequest: boolean     // controls SpecificDate tab visibility
  leavingInCfg: number[]            // NOT used for UI — leavingIn from ticket-info is used instead
}
```

**Rule:** `leavingIn` for the dropdown always comes from `ticketInfo.config.leavingIn`.
`allowMinutesRequest` / `allowDateTimeRequest` always come from `GuestCfgState.guestConfig`.

---

## TicketInfo shape (key fields)

```
TicketInfo {
  company: { uuid, name, operatorId }
  config: GuestConfig           ← allowRequest, allowPayment, allowBackAndForth, leavingIn
  guest?: { fullName, make, arrivalTime, phoneNumber, countryCode }
  location: { uuid, id, hasSurvey, ... }
  ticket?: { ticket, totalToPay, requested, expectedCheckout, hasTip, compensation, temporalCheckout, validation }
  rate?: { id (RateType), price, partnerProcessPayments }
  payments?: Payment[]
  validationInfo?: { amountCalculated }
}
```

---

## Ticket page template rendering logic

```
<app-header />
<rate-info />              — shown when rate.price > 0 AND rate is not Overnight AND no compensation
<check-in-time />          — always shown (displays guest.arrivalTime)
<phone-number-form />      — shown when location.hasSurvey AND guest.phoneNumber === null
<request-car />            — shown when config.allowRequest AND (allowMinutesRequest OR allowDateTimeRequest) AND not temporalCheckout
<ticket-actions />         — shown when ticket is not temporalCheckout
<qr-code-container />      — always shown
```

If `ticket.temporalCheckout` is true → skip request-car, ticket-actions, show "See you soon!" instead.

---

## Phone number section

`PhoneNumberState.hasPhoneNumber` computed:
- If `location.hasSurvey`: returns `true` if `guest.phoneNumber` is set, else requires 10-digit manual input
- If no survey: always `true`

The "Request car" button and "Add Tip" button are disabled while `!hasPhoneNumber()`.

---

## Request car flow

### Time selection UI
- Tabs shown only if `GuestCfgState.guestConfig().allowMinutesRequest` or `allowDateTimeRequest`
- Minutes dropdown options come from `ticketInfo.config.leavingIn`
- If both tabs enabled → select shown; if only one → that section shown directly
- Guest Profile present → time selection hidden entirely (`hasGuestProfile` gates the form)

### `onRequestCar()` in `TicketActionsComponent`
- Payload: `{ operatorCompanyId, parkingLocationId, ticket, phoneNumber: guest.phoneNumber ?? '', minutes }`
- POST `/partner/guest/request-car`
- On success: sets `isCarRequested = true`, `expectedCheckout = result.expectedCheckout`
- On 409 with code `VEHICLE-REQUEST-CAPACITY-LIMIT-REACHED`: opens `CapacityLimitDialog`
- Tip prompt dialog shown for specific location UUIDs after request

### Cancel request
- `onCancelRequest()` → PUT `/partner/guest/cancel-request`
- On success: sets `isCarRequested = false`

---

## B&F SMS workflow (backend-triggered)

When a car request is received by the backend with a phone number from the Guest Profile:
1. Backend sends SMS to guest confirming request received
2. All subsequent coordination (timing, delivery) is handled via SMS (Back & Forth)
3. Frontend shows static "CAR REQUESTED" state

**Open question (2026-05-20):** In B&F flow, does the backend return `expectedCheckout`? If not, the post-request state needs an SMS confirmation message instead.

---

## `partnerProcessPayments` flag on `Rate`

`rate.partnerProcessPayments: boolean` controls whether Overnight rate blocks payment.

- `false` (valet-collected) → payment is allowed; PAY button visible; `paymentGuard` passes
- `true` (partner collects) → payment blocked; `OvernightRateDialog` shown; guard redirects

This flag must be checked in **three places** (all updated):
1. `paymentGuard` — `payment.ts`
2. `displayPayButton` computed — `ticket-actions.component.ts`
3. `onClick()` in `pay-button.component.ts`

---

## Payment flow (summary)

Gateway set per-location from `TicketActionsState.configuration.paymentGateway` (loaded by `ticket-actions` via `getGuestPageConfiguration()`).
`displayPayButton` requires: gateway is non-null, price > 0, no compensation, no temporalCheckout, not Overnight+partnerProcessPayments, `allowPayment = true` (from GuestCfgState).

Full details in `project_payment_flows.md`.

---

## Windcave cross-tab communication

After payment, popup writes `WINDCAVE_STATUS` to `localStorage`.
`TicketComponent` listens via `@HostListener('window:storage')` → hides loader, closes dialogs, shows snackbar (Approved / Declined / Cancelled).

---

## Key component locations

| Component | Path |
|---|---|
| `TicketComponent` | `pages/ticket/ticket.component.ts` |
| `RequestCarComponent` | `shared/components/request-car/` |
| `TicketActionsComponent` | `shared/components/ticket-actions/` |
| `PhoneNumberFormComponent` | `shared/components/phone-number-form/` |
| `CheckInTimeComponent` | `shared/components/check-in-time/` |
| `RateInfoComponent` | `shared/components/rate-info/` |
| `QrCodeContainerComponent` | `shared/components/qr-code-container/` |
| `PayButtonComponent` | `shared/components/pay-button/` |
| `RequestCarDialogComponent` | `shared/components/request-car-dialog/` |

---

## Key services

| Service | Path | Methods |
|---|---|---|
| `PartnerService` | `core/services/partner/` | `getTicketInfo`, `requestCar`, `cancelRequest`, `getPaymentDetail`, `makePayment` |
| `LocationDetailService` | `core/services/location-detail/` | `getGuestPageConfiguration`, `getGuestPageCfg` |
| `LoggerService` | `core/services/logger-service/` | Bugfender wrapper |
