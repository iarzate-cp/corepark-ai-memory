---
name: Feature — Check-In Guest Profile & B&F SMS
description: Context, decisions, and open items for the check-in Guest Profile + SMS Back & Forth feature on branch feature/check-in
type: project
originSessionId: 21ac8f31-9b7e-4189-9c9a-3120a04e38bc
---
## Context

Branch: `feature/check-in`

**Product requirement (from boss):**
When a guest has a Guest Profile (phone number stored at check-in), the guest page should skip the minutes selection UI and show only a "Request car" button. After the request, the backend activates the SMS Back & Forth (B&F) workflow to coordinate car delivery — no timing is handled on the frontend.

**Jorge (backend) clarified:**
> "So what's needed is for B&F to be triggered when a car is requested from the guest page. CorePark receives the alert and sends the car request. CorePark activates the B&F workflow by sending the customer an SMS confirming that the request was received, and from there on, all communication is handled via SMS."
> 
> Jorge checked on BE: "I think we already had something like that, let me check on BE, but for now, it looks doable!"

---

## What was implemented (2026-05-20)

### `shared/components/request-car/request-car.component.ts`
Added computed:
```typescript
readonly hasGuestProfile = computed(() => !!this.ticketInfo()?.guest?.phoneNumber)
```

### `shared/components/request-car/request-car.component.html`
Wrapped the entire time-selection form (type toggle + minutes dropdown + date picker) in:
```html
@if (!hasGuestProfile()) { ... }
```
The "CAR REQUESTED" state block is NOT gated — it still shows after a successful request in both flows.

### What was NOT changed (already worked)
- `TicketActionsComponent.#requestCar()` already uses `ticketInfo?.guest?.phoneNumber ?? ''` as the phone number in the payload
- `PhoneNumberState.hasPhoneNumber` already returns `true` when `guest.phoneNumber` is set, so the button is never disabled
- `RequestCarState.selectedMinutes` defaults to `leavingIn.at(0)` — the request payload still has a valid `minutes` value even without UI selection
- `displayPhoneNumberSection` in `TicketComponent` already hides the manual phone form when `guest.phoneNumber` is set

---

## Open item — post-request state in B&F flow

**Status: pending confirmation from Jorge (as of 2026-05-20)**

Currently, after a successful request, `<request-car>` shows:
```
CAR REQUESTED
Your car was requested, the estimate delivery time is [expectedCheckout]
```

In the B&F flow, timing is coordinated via SMS — the `expectedCheckout` concept may not apply.

**Question for Jorge/backend:** Does the `POST /partner/guest/request-car` endpoint return `expectedCheckout` when B&F is triggered, or should the frontend show a different message (e.g., "You will receive an SMS confirmation shortly")?

**If backend does NOT return expectedCheckout in B&F flow:** update the `@if (isCarRequested())` block in `request-car.component.html` to conditionally show SMS message vs. time estimate based on `hasGuestProfile()`.

---

## Flow summary

```
Guest arrives → scans QR / opens link
  ↓
TicketPage loads → GET /partner/guest/ticket-info
  ↓
guest.phoneNumber present?
  ├── YES (Guest Profile): hide minutes form → show only "Request car" button
  │     ↓
  │   POST /partner/guest/request-car  { phoneNumber: guest.phoneNumber, minutes: leavingIn[0] }
  │     ↓
  │   Backend triggers B&F SMS workflow → guest receives SMS confirmation
  │   Frontend shows "CAR REQUESTED" (expectedCheckout display TBD)
  │
  └── NO (no Guest Profile):
        Show minutes/date selection UI → user picks time
          ↓
        POST /partner/guest/request-car  { phoneNumber: '', minutes: selected }
          ↓
        Frontend shows "CAR REQUESTED" + expectedCheckout time
```
