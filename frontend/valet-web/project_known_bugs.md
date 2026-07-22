---
name: Known Bugs
description: Active bugs in the codebase that haven't been fixed yet
type: project
originSessionId: 1a79ed89-b142-4110-a666-1033820e8843
---
## No known bugs at this time.

### Resolved

**SettingsService — missing `environment.endpoint`** (fixed 2026-05-12)
All 4 HTTP methods in `settings-service.ts` now prepend `${environment.endpoint}` to their paths.
