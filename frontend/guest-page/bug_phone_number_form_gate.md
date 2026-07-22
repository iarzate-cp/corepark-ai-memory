---
name: Phone number form gate bug (July 2026)
description: Backend stopped sending guest.phoneNumber when absent; frontend used strict === null check and broke — fixed by aligning to truthy check
type: project
originSessionId: c6688e13-caaf-4bd4-9f6c-4e0fc1140b52
---
Bug found on 2026-07-02 on `main`. Location with `hasSurvey: true` and no guest phone left users stuck: the phone-number-form was hidden but the Request Car / Pay buttons stayed disabled — no way to unblock from the UI.

**Root cause — divergent checks for the same domain question:**
- `pages/ticket/ticket.component.ts` displayPhoneNumberSection used strict `guest?.phoneNumber === null`.
- `core/states/phone-number.ts` hasPhoneNumber used truthy check `if (guest.phoneNumber)`.

The ticket-info endpoint used to send `phoneNumber: null` explicitly when absent, so both checks agreed. Around mid-2026 the backend started **omitting the property** entirely (likely a `@JsonInclude(NON_NULL)`-style serialization change). `undefined === null` is `false` → form hidden; `if (undefined)` is falsy → buttons disabled. Deadlock.

The frontend `Guest` type (`core/models/definitions/common.ts:46`, `ticket.ts:29`) had always typed it as `phoneNumber?: string` — the backend change was legal per the contract; the frontend's `=== null` assumption was stricter than the type.

**Fix:** changed `ticket.component.ts:86` to `!ticketInfo?.guest?.phoneNumber` so both call sites use the same truthy check. Now works whether backend sends `null`, `undefined`, or omits the property.

**Why:** the frontend files involved had not been touched in ~17 months (`ticket.component.ts` refactored 2025-01-20 in commit `420a6fbf`; `phone-number.ts` last touched 2025-03-05 in `13a6b2af`). The regression came entirely from the backend side.

**How to apply:** when reading optional fields from any ticket-info / guest-cfg response, prefer truthy checks over strict `=== null`. The backend in this project treats optional properties as omit-when-absent, not send-as-null. If you see a `=== null` guarding an optional field somewhere else, it's likely another latent instance of the same bug.
