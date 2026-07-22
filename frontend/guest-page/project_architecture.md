---
name: Architecture Patterns — frontend-guest-page
description: Key structural and architectural decisions: standalone, signals, interceptors, initializers, theming
type: project
originSessionId: 5af14dee-5704-4786-bda3-2b912b2b7349
---
All architectural decisions in this project follow these patterns. Understand them before adding any new code.

**Standalone-first:** No NgModules. Every component has `standalone: true` with explicit `imports[]`.

**Signal-first state:** Global state lives in `@Injectable({ providedIn: 'root' })` services using `signal()`, `computed()`, and `effect()`. No RxJS BehaviorSubject for state. Files are in `src/app/core/states/`.

**Functional HTTP interceptors:** Applied via `withInterceptors()` in `app.config.ts`. Files: `headers.ts`, `body.ts`, `identifiers.ts`, `recaptcha.ts`, `logger-interceptor.ts`.

**APP_INITIALIZER hooks:** Six initializers run before bootstrap to restore localStorage cache and load SDKs (Bugfender, FreedomPay script, reCAPTCHA, ticket info, URL params, location config). Files in `src/app/core/initializers/`.

**localStorage persistence:** `TICKET_INFO`, `URL_PARAMETERS`, `CONFIGURATION`, `WINDCAVE_STATUS` are all persisted and restored on load. Utility: `src/app/core/utils/storage.ts`.

**Multi-payment-gateway:** Gateway selected per-location from `GuestPageConfigState`. Enum: `PaymentGateway` (`SQUARE | WINDCAVE | FREEDOMPAY`). Each has its own flow — see `docs/payment-flows.md`.

**Dynamic theming:** Per-location CSS variables. Two base themes: `corepark` and `curbstand`. Logic in `utils/theme-utils.ts` and `utils/location-themes.ts`.

**Lazy routing:** All page routes use dynamic `import()`. Route guard `paymentGuard` protects `/payment`.

**Path aliases:** All imports use `@components/`, `@services/`, `@states/`, `@enums/`, etc. — never relative paths across feature areas. Defined in `tsconfig.json`.

**RateType enum:** All rate type IDs must use `RateType` from `@enums/rate-type` — never hardcode magic numbers. Values: Flat=1, Fixed=2, Variable=3, Temporary=4, Overnight=6. Not all rate types support payment (e.g. Overnight is blocked at the frontend level).

**pnpm peer deps:** pnpm does not expose transitive dependencies. Any package used directly in source must be listed as a direct dep in `package.json`, even if it is a peer dep of another package (e.g. `@zxing/library`, `@zxing/browser` are peer deps of `@zxing/ngx-scanner` but must be declared explicitly).

**Pay button visibility:** `displayPayButton` in `ticket-actions.component.ts` requires `configuration.paymentGateway` to be non-null. This comes from a separate `getGuestPageConfiguration()` call. If a location has no payment gateway configured in the backoffice, that API returns no `configuration` object and the button is correctly hidden. Always verify backoffice gateway setup before debugging frontend visibility.

**How to apply:** Any new feature must follow standalone + signals pattern. No new NgModules, no new class-based interceptors, no BehaviorSubject for global state.
