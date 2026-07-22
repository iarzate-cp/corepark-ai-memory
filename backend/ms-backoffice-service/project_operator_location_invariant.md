---
name: Operator is the root; location always resolves to an operator
description: Data model invariant — every location has an operator FK, so any query starting from a location can rely on operatorUuid being present. Clarifies why defensive null checks on operatorUuid aren't required.
type: project
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
Corepark's data model treats `operator_company` as the top-level tenant.
When a user registers, an operator record is created automatically and their
identifier is bound to that operator. Every `parking_location` row is
FK-bound to `operator_company` — there's no orphan location.

Practical consequence: when the backend resolves a location (by id, by uuid,
by whatever), the operator identifier is always resolvable via JOIN. There's
no need to guard for a missing `operatorUuid` in production correctness — if
you have a valid location, you have its operator.

**Why:** Israel clarified this on 2026-07-17 after we shipped the
vehicle-info GET in ms-valet-service (`com.corepark.valet.vehicleinfo`) that
returns `operatorUuid` in the payload. We had a defensive
`if (!operatorUuid) return EMPTY` in HeaderImageService for backward compat
with the state-based caller path (`TicketInfoState` may be null while
loading), but for the new arg-based caller path, `operatorUuid` is guaranteed
by the BE contract.

**How to apply:**
- Backend queries starting from a location can safely `JOIN
  company.operator_company` — the match is guaranteed, no LEFT JOIN needed.
- Frontend API payloads that resolve from a location should type
  `operatorUuid: string` (not `string | null`).
- Existing defensive null checks on `operatorUuid` can stay for older paths
  that pull from a state signal (which may be null during load), but new
  paths that receive it explicitly can trust it non-null.
- Contrast with `guest.*` fields (firstName/lastName/phoneCode/phoneNumber)
  and `vehicleInfo.*` fields, which ARE legitimately nullable — those depend
  on what the operator captured at check-in, not on the data model core.
