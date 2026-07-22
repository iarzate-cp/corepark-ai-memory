---
name: Destructive confirmations should embed the full detail of the item being removed
description: Delete/archive dialogs must show the user what they're about to lose — not just the name. Reuse a shared detail component so the confirmation is a decision, not a leap of faith.
type: feedback
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
**Rule:** When building a destructive confirmation dialog (delete, archive, revoke, permanent action), **embed a detail view of the target item** inside the dialog body. Don't rely on the user remembering what "Delete rate: $0 Overnight" means.

**Why:** The user asked to add the rate detail inside the delete dialog (2026-07-09) after seeing the confirmation was just a message + checkbox. Their concern: heavy-usage feature + easily-confused rate names = high risk of deleting the wrong thing. The fix is embedding the same detail view the user would see if they clicked "View detail". This turned a leap of faith into an informed decision.

**Extended pattern (worth applying to other modules):**

Destructive confirmations follow this structure:
1. **Warning message** — plain-english statement of what's about to happen. Use `<strong>` for the item name so it stands out.
2. **Full detail of the target** — same read-only view the user has for browsing. Reuse a shared presentational component (`<foo-detail-panel [foo]="target">`) so single source of truth.
3. **Confirmation gate** — a checkbox with a label like "I understand" that unlocks the destructive button. Uses `[extraActions]="true"` + `<cp-checkbox extra-action>` slot on `cp-dialog-content`.
4. **Destructive button** — red styling (custom `.danger-btn` — `cpButton` has no `danger` variant). Disabled until checkbox is checked.
5. **Cancel** — provided automatically by `exitLabel="Cancel"` on `cp-dialog-content`. Don't add a manual Cancel button.
6. **Dialog owns the HTTP call** — success → `NotificationService.show({ type: 'success' })` + `DialogRef.close(true)`. Error → `notifyHttpError` + dialog stays open (allows retry). The page listens to `.afterClosed()` and reloads on `true`.

**How to apply:**

For any new destructive dialog:
- If a detail panel component already exists → reuse it as `<x-detail-panel [x]="target" />`
- If no detail panel exists → extract one **before** building the confirmation dialog. The panel becomes reusable across (a) view detail dialog, (b) delete confirmation, (c) any future flow like archive/duplicate/version-history.
- The panel is presentational only: `input.required<X>(...)`, no injects other than pipes.

**Reference implementation:** `src/app/shared/components/rate-delete-dialog/` in `frontend-commerce` consumes `src/app/shared/components/rate-detail-panel/`. The same panel is used by `rate-detail-dialog`.

**Never:**
- Ship a delete dialog that only shows the item name and expects the user to remember what they're deleting
- Duplicate the detail rendering between the "view" and "delete" dialogs — extract to a shared panel
- Skip the confirmation checkbox for high-impact deletes
- Auto-close the dialog on error — the user needs to see the error and retry
