---
name: Unimplemented — Guest Page phone should trigger checkout SMS (survey link)
description: Reported 2026-07-02 as a NOT-YET-IMPLEMENTED behavior (not a bug). Today the checkout SMS only fires when the phone came from the Valet App path. The Guest Page → checkout SMS wiring is a gap to build, not a regression to fix
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
## Reported gap (2026-07-02)

Israel clarified this is **NOT a bug** — it is a behavior that has never been implemented:

1. If the phone number is entered in the **Valet App** → the checkout SMS sends (and carries the survey link when a survey is active).
2. If the phone number is entered in the **Guest Page** → the checkout SMS does NOT send.
3. **Expected end state:** both entry points should end in the same checkout SMS (and therefore the same survey trigger).

Framing matters: nothing is broken. The Valet App path was built end-to-end; the Guest Page path was not. Any work here is a feature build, not a regression fix.

## What exists today

### Valet App path (works end-to-end)

- Phone written to `company.parking_service.phone_number` + `phone_code_id` at check-in via `ValetDao.QUERY_CHECK_IN` (INSERT) or mid-service via `TicketsDao` (UPDATE at `TicketsDao.java:75`).
- On `POST /v2/check-out`, `ValetDao.QUERY_GET_SMS_CHECKOUT_INFO` (`ValetDao.java:358-376`) reads `phoneCode` + `phoneNumber` back.
- `ValetService.java:1476-1484` composes the SMS and sends via `notificationService.sendSms` (Feign → ms-notifications-service).
- Survey link is appended by `SurveyService.handleSurvey` (see `project_survey_trigger_flow.md`).

### Guest Page today

- `POST /partner/guest/request-car` (`GuestPageController.java:67-89`) does UPDATE `parking_service.phone_number` + `phone_code_id` via `CommonDao.QUERY_UPDATE_PHONE_NUMBER` when the guest provides both `phoneNumber` and `countryCode` in the car-request payload (`GuestPageService.java:124-146`).
- **No other Guest Page endpoint persists a phone.** `POST /partner/guest/payment/pay-ticket` doesn't touch phone columns. `POST /partner/guest/cancel-request` doesn't either.
- **No wiring exists** to route a Guest-Page-collected phone into the checkout SMS pipeline outside of that request-car UPDATE.

## Why "same table, same columns" isn't enough

On paper, the request-car UPDATE writes to the exact columns the checkout SELECT reads from, so it could look like it should work today. In practice it doesn't (per the team). Before designing the implementation, we need to nail down which of the following describes reality:

- **A. The request-car UPDATE never runs in the failing scenario** — e.g., the customer entered the phone on a different Guest Page screen (payment form) that doesn't call request-car at all. This is the most likely story if Israel says "not implemented" — the intended UX is "guest enters phone anywhere on Guest Page → SMS sends," but the only wiring today is on request-car.
- **B. The request-car UPDATE writes to a row the checkout SELECT can't match** — check_in mismatch, older cycle, `Long.parseLong` blow-up on formatted input. Would look like an implementation defect on the request-car path specifically.
- **C. Both A and B** — request-car works for its narrow flow, but the general "Guest Page phone" concept isn't implemented across all Guest Page entry points.

Confirming A vs B changes the implementation scope substantially, so this is the first thing to pin down when we pick this up.

## Concrete DB queries to characterize current state

```sql
-- 1) After a repro (customer entered phone in Guest Page, checkout done, no SMS)
SELECT ticket, check_in, check_out, phone_number, phone_code_id
FROM company.parking_service
WHERE operator_company_id = <op> AND parking_location_id = <loc>
  AND ticket = '<ticket>'
ORDER BY check_in DESC;
-- If phone_number IS NULL on the checked-out row → scenario A (nothing persisted the phone)
-- If phone_number IS NOT NULL → scenario B (persisted but not read; investigate check_in match)

-- 2) Did the survey petition get created despite the SMS not going out?
SELECT id, uuid, phone_number, survey_id, created_at
FROM company.survey_petition
WHERE operator_company_id = <op> AND parking_location_id = <loc>
  AND ticket = '<ticket>'
ORDER BY created_at DESC;
-- No row → handleSurvey was never called → the SMS composition itself was skipped upstream

-- 3) Is a survey active for this location? (rule this out first)
SELECT id, deactivated_at FROM company.survey
WHERE operator_company_id = <op> AND parking_location_id = <loc>;
```

## Implementation shape (once scenario is confirmed)

### If scenario A (most likely)

The Guest Page needs an explicit "capture phone" flow that persists to `parking_service` regardless of whether the guest is requesting the car, paying, or just registering contact info. Options:

- **A1.** New endpoint `POST /partner/guest/phone` on ms-valet-service that just UPDATEs `phone_number` + `phone_code_id` on the active row. Frontend calls it whenever the guest submits a phone from any screen. Simplest.
- **A2.** Piggyback on the payment flow: extend `TicketPaymentRequestBean` to optionally include phone, and have `payTicket` persist it if provided. Ties phone capture to payment, which may not be UX-correct.
- **A3.** Server-side event: subscribe to the Guest Page's existing contact-capture events (if any) and update the row from there.

A1 is the cleanest — no coupling of "phone capture" to any other Guest Page domain action.

### If scenario B

Fix the mismatch at the write or read site:

- Sanitize `CommonDao.updatePhoneNumber` input — strip non-digits before `Long.parseLong` (`CommonDao.java:182`) to avoid silent failures on formatted phones like `+1 555-1234`.
- Consider dropping `AT TIME ZONE 'UTC'` from `QUERY_VALID_IF_HAS_CHECK_IN` (`CommonDao.java:27-28`) so the check_in stays as `timestamptz` throughout — reduces the risk of the guest write and the valet checkout keying on different values.
- Or, more robustly, make the checkout SMS SELECT key on `check_out IS NULL` (post-UPDATE, `check_out IS NOT NULL`, so change the ordering) rather than exact `check_in` equality. This requires rethinking the query since QUERY_CHECKOUT already fires before it.

### Cross-cutting concerns for either scenario

- **Feature flag / gating:** the survey link is already gated by the `company.survey` active-row check. If the Guest Page path lands, the same admin CRUD on/off remains the toggle.
- **Idempotency:** if the guest submits phone twice, we already UPDATE (not INSERT into `survey_petition`) — but `SurveyService.handleSurvey` runs on checkout, not on phone capture, so no double-petition risk.
- **Consent:** confirm with product whether entering a phone via Guest Page constitutes SMS opt-in. Valet App capture already implies opt-in per current UX; Guest Page may need explicit consent copy.

## Related memory

- `project_survey_trigger_flow.md` — the full end-to-end trigger flow (as-built)
- `project_survey_petition_lifecycle.md` — why petitions are created by valet, not backoffice
- `project_survey_progress_snapshot.md` — the admin CRUD we shipped, which is the on/off switch feeding this trigger

## Status

Not scheduled. Documented for future pickup when the team prioritizes closing the Guest-Page-phone → checkout-SMS gap.
