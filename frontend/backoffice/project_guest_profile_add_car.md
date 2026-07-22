---
name: Guest Profile — Add Car feature
description: Implementación del flujo para agregar un vehículo a un guest profile existente
type: project
originSessionId: 1990a33e-d5e2-4342-aba1-a45cbdd6ebbb
---
Branch: `hotfix/guest-profile/add-car`

Feature completa e integrada. Cambios realizados:

1. **`profiling.d.ts`** — Agregados `CreateCarProfileRequest` y `CreateCarProfilePayload`
2. **`profiling.service.ts`** — Nuevo método `createCarProfile(payload)` → `POST /backoffice/car-profiles`
3. **`shared/components/guest-profile-add-car/`** — Nuevo dialog (ts + html + scss + imports + index)
4. **`guest-profile-cars`** — Integrado: método `onAddCar()` + botón "Add Car" en el header

**Why:** El `ProfilingService` no tenía el método de creación de autos aunque el endpoint ya existía en el backend.

**How to apply:** Commiteado. Pendiente PR contra `main`.

### Detalles no obvios

- El endpoint `POST /backoffice/car-profiles` espera una **lista**, no un objeto individual: `{ carProfileRequests: [...] }`
- `vin` es **opcional** en create (a diferencia del update donde el form lo trata como required)
- `hasDamage` va hardcodeado en `false` — el flujo de photo reel está pendiente de implementar
- El dialog no usa `MAT_DIALOG_DATA` — lee `guestProfileId` directo de `ProfilingState.guestProfile()`
- `disabled` en `guest-profile-cars` solo afecta delete (cuando queda 1 auto); el botón Add no se deshabilita
