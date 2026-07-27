---
name: Mock como referencia, no spec — verificar backing antes de implementar
description: Cuando un mock introduce controles sin soporte de backend o desarrollo previo, marcarlos como pendientes de confirmación en lugar de construirlos silenciosamente
type: feedback
originSessionId: 5506e125-b5d9-4d50-a707-64609419d5d0
---
Cuando un mock muestra toggles, opciones o controles que no aparecen en el desarrollo previo ni tienen soporte del backend, NO los implementes silenciosamente. Márcalos explícitamente como "mock-only, requiere confirmación de producto/backend" y para el flujo.

**Why:** En el rediseño de activity-by-rate-class, el mock introdujo:
- **AM/PM toggle** en el card horario (nunca en codebase, sin backend, no discutido).
- **Monthly** granularity (enum `ActivityGranularity` del backend no tiene MONTH).
- **Custom** granularity (semantic ambigua — ya existe el date picker para rangos arbitrarios).
- **Dropdown en el card horario** (endpoint `/hourly-profile` no toma granularity).

Implementé el granularity dropdown en ambas cards (incluyendo la inútil del horario) y listé AM/PM como "pendiente del mock" tres turnos seguidos antes de que Israel empujara: "yo no recuerdo que hubiéramos aplicado AM/PM en alguna parte del desarrollo". Correcto — nada de eso tenía contexto previo. Terminamos removiendo el dropdown del hourly y aceptando que Monthly/Custom se agregan cuando backend + producto lo definan.

**How to apply:**
- Antes de implementar cualquier control tomado de un mock, cross-checar contra:
  - Backend endpoint definitions (`@definitions/`, servicios en `core/services/`).
  - Estado previo del código (git log, memoria existente).
  - Historia del producto si se conoce.
- Si no hay backing, listar el ítem explícitamente como "mock-only, needs product/backend confirmation" y NO implementar hasta confirmar.
- Cuando Israel dice "no recuerdo haberlo aplicado" sobre algo del mock, es señal fuerte de que el mock carece de contexto. Priorizar backend contract + historia sobre mock UI.
- Cuando un mock muestra 4 opciones (Daily/Weekly/Monthly/Custom) pero backend solo soporta 3 (DAY/WEEK/HOUR), matchear al backend, no al mock. Si el mock tiene autoridad, exigir alineación con backend primero.
