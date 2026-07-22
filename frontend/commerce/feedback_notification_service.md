---
name: Use NotificationService (DS), not MatSnackBar, for new feedback
description: New features should use @corepark/corepark-ui NotificationService for toasts/alerts. MatSnackBar coexists in legacy code but should not be added to new files
type: feedback
---

**Rule:** for user feedback (success, error, warning, info) in **new code**, use `NotificationService` from `@corepark/corepark-ui`. Do NOT reach for `MatSnackBar` from Angular Material even though many legacy files still use it.

**Why:** Israel called this out on 2026-07-01 after we shipped the survey MVP using MatSnackBar. The DS has a first-class notification service (`cp-alert` for inline banners, `NotificationService` for toasts) — using Material for feedback undermines the design system's role as the visual source of truth. Migrated the whole survey flow on the same day.

## How to apply

**Setup (once per app)** — add to `app.config.ts`:
```ts
import { provideNotifications } from '@corepark/corepark-ui'

providers: [
    provideNotifications({ position: 'bottom-right' }),
    // ...
]
```
Positions: `'top-left' | 'top-center' | 'top-right' | 'bottom-left' | 'bottom-center' | 'bottom-right'`.

**Usage in a component:**
```ts
import { NotificationService } from '@corepark/corepark-ui'

readonly #notif = inject(NotificationService)

// Success
this.#notif.show({ type: 'success', title: 'Basic survey created' })

// Error (with description)
this.#notif.show({
    type: 'error',
    title: 'Could not create survey',
    description: 'Server returned 500',
})

// Persistent (no auto-dismiss)
this.#notif.show({ type: 'warning', title: 'Unsaved changes', duration: 0 })
```

**Types:** `'info' | 'success' | 'error' | 'warning'` (note: `'error'`, NOT `'danger'` — that's `cp-alert`'s variant name).

**Duration:** default `4000` ms. Pass `0` for persistent.

## For HTTP errors — use the helper

We built `@utils/notify.ts` with `notifyHttpError(notif, error, title)` that:
- Maps HTTP status to notification type: `>= 500 → 'error'`, `< 500 → 'warning'`
- Extracts server-provided message from `err.error.message` or `err.error.error_description`
- Falls back to `"An unexpected error has occurred"` for 5xx or missing message

Usage:
```ts
import { notifyHttpError } from '@utils/notify'

.subscribe({
    error: (err: HttpErrorResponse) =>
        notifyHttpError(this.#notif, err, 'Could not load survey'),
})
```

## Naming convention for titles

- Success: **"X created"** / **"X updated"** / **"X saved"** — short, past tense, subject-object clear
- Error: **"Could not X"** — neutral, doesn't accuse ("Failed to X" reads harsher). Description carries specifics.

## For inline banners (not toasts)

Use `cp-alert` instead — variants `'info' | 'success' | 'danger' | 'warning'` (note the `'danger'` naming difference from `NotificationService`), `title`, `dismissible`, `(dismissed)` output. Renders inline in the component tree, not overlay.

Rule of thumb: **toast (`NotificationService`)** for transient feedback after an action; **`cp-alert`** for persistent warnings in the middle of a form or page.

## Migration approach

Do NOT bulk-migrate legacy MatSnackBar usages — coexistence is fine. Migrate on touch: if you're editing a file that uses MatSnackBar for feedback, swap it for `NotificationService` in the same change.
