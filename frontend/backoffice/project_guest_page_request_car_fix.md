---
name: frontend-guest-page — fix merge conflict request-car (develop)
description: Corrección de merge conflict en develop que ocultaba opciones de Minutes/SpecificDate
type: project
originSessionId: 28fcfc49-958f-4478-af5a-24c5693e47e1
---
Merge conflict resuelto en `develop` de `frontend-guest-page` (branch `feature/allow-requests` → `develop`).

**Why:** El merge introdujo `@if (!hasGuestProfile())` wrapeando todo el bloque de opciones en `request-car.component.html`. Como `hasGuestProfile = !!ticketInfo?.guest?.phoneNumber`, cualquier ticket con número de teléfono registrado dejaba de ver las opciones de minutos/fecha — aunque `allowMinutesRequest` y `allowDateTimeRequest` estuvieran en `true`.

**How to apply:** Si vuelve a haber un merge en este archivo, verificar que `!hasGuestProfile()` no envuelva las opciones de request. La condición correcta es solo `@if (allowRequest() && !isCarRequested())`.

## Resolución aplicada

- Se eliminó el wrapper `@if (!hasGuestProfile())` del template
- Se mantuvieron las condiciones del feature branch: `requestOptions().includes(requestCarType().Minutes/SpecificDate)`
- Se mantuvo `ticketInfo().config.leavingIn` (no `leavingInCfg`) — consistente con el TS del componente
- Archivo: `src/app/shared/components/request-car/request-car.component.html`

Commit pendiente de hacer (texto: `fix: remove hasGuestProfile guard blocking request car options`).
El merge commit también está pendiente (texto: `chore: resolve merge conflict in request-car template`).
