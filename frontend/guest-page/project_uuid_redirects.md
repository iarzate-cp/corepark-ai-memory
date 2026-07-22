---
name: UUID Redirect — Mistyped Printed UUID
description: A location UUID was mistyped on printed material (missing leading zero); redirect added in uuidSetter()
type: project
originSessionId: 329117be-b4bf-405e-ab65-c965cc1ab1dd
---
A customer's location UUID was printed incorrectly — missing the leading zero:

- **Wrong (printed):** `7f64fe6-61a5-4e8d-9638-32ed1a73e6d0`
- **Correct:** `07f64fe6-61a5-4e8d-9638-32ed1a73e6d0`

Fix: added a mapping in `core/utils/uuid.ts` inside `uuidSetter()` (line ~135) that returns the correct UUID when the mistyped one is received. A comment explains the reason (`// UUID was mistyped on printed material — missing leading zero`).

**Why:** Physical printed material cannot be updated retroactively; the redirect must live in the app.

**How to apply:** If another similar misprint is reported, add a new `if` block in `uuidSetter()` following the same pattern, with a comment explaining it's a printed-material typo.
