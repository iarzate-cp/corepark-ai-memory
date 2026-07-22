---
name: Env-driven cross-app URL host in frontend-commerce
description: When frontend-commerce needs to build a URL that points to a DIFFERENT frontend (e.g., guest page), add a host key in each environment.*.ts and a small utility that reads it
type: reference
originSessionId: b094aa38-29e6-42a7-bdf0-66c9d0e65af9
---
# Env-driven cross-app URL host

When `frontend-commerce` needs to construct a URL that opens **another frontend** (e.g., the guest page), do NOT hardcode the host — the same code has to produce `https://localhost:4100`, `https://dgnw8qm5ijo58.cloudfront.net`, and `https://guest.corepark.com` depending on the build.

## Pattern (as of 2026-07-17)

1. Add the host key to each of the three env files, next to the existing keys (`endpoint`, `operatorsMediaHost`, `googleKey`, …):
   - `src/environments/environment.ts`      → `guestPageHost: 'https://localhost:4100'`
   - `src/environments/environment.dev.ts`  → `guestPageHost: 'https://dgnw8qm5ijo58.cloudfront.net'`
   - `src/environments/environment.prod.ts` → `guestPageHost: 'https://guest.corepark.com'`

2. Add a small pure builder in `src/app/core/utils/`:
   ```ts
   // vehicle-info-url.ts
   import { environment } from '@env'
   export const vehicleInfoUrl = (locationUuid: string, ticket: number): string =>
     `${environment.guestPageHost}/vehicle-info/${locationUuid}/${ticket}`
   ```

3. Import via the `@utils` alias.

## Reusable pattern

If a new consumer needs another cross-frontend URL, follow the same recipe: add a `xxxHost` key in all three env files + a small utility that uses it. Do not put the string literal directly in a component.

## Related

- `project_vehicle_info_ecosystem.md` — the feature that introduced this pattern.
- `reference_related_repos.md` — canonical FE paths and their host mapping.
