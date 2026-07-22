---
name: Firebase angular-fire Workaround
description: Known bug in @angular/fire@20 + firebase 11.5+ and the applied fix
type: project
originSessionId: 1a79ed89-b142-4110-a666-1033820e8843
---
`@angular/fire@20.0.1` has bug #3642: `provideFirebaseApp(() => initializeApp(config))` doesn't correctly register the app in Firebase's component registry when using `firebase >= 11.5.0`. Caused two errors: "No Firebase App '[DEFAULT]'" and "Service database is not available".

**Root cause:** With pnpm's module resolution, dual firebase versions (11.x and 12.x) created two separate `@firebase/app` instances with isolated component registries. Registering the database in one registry while the app lived in another broke `getDatabase(app)`.

**Fix applied (in `app.config.ts`):**
1. pnpm override in `package.json` forces single `firebase@11.10.0` across all deps.
2. Module-level `const app = initializeApp(config)` passed explicitly to `provideAuth` and `provideDatabase`.

**When to remove:** When `@angular/fire@21` reaches stable release, restore the standard pattern: `provideFirebaseApp(() => initializeApp(config))`, `provideAuth(() => getAuth())`, `provideDatabase(() => getDatabase())`.

**Why:** Documented workaround for angular-fire issue #3642. Applied 2026-05-12.
**How to apply:** Do not change `app.config.ts` Firebase setup unless upgrading to @angular/fire@21 stable.
