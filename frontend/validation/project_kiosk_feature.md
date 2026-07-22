---
name: Kiosk feature — estado y decisiones no obvias
description: Feature de kiosco (self-scan de ticket) vive en frontend-validation en /kiosk; estado al 2026-07-17 tras merge a develop
type: project
originSessionId: 6e00c961-e831-4186-bb1d-d0ae8fb61c3b
---
# Kiosk feature

Feature de kiosco físico donde el huésped escanea su ticket y dispara un request-car. Vive en **frontend-validation** (no en frontend-guest-page, como pudiera parecer por ser una pantalla no-operador).

## Estado (2026-07-17)
- Rama: `feature/kiosk-scan-module` (superset de la vieja `feature/scan-module`, que quedó abandonada).
- **Merge a `develop` completado** (commit `6a50469`), sin PR a `main` todavía.
- **Why:** La rama estaba huérfana desde marzo 2026; se retomó porque el usuario notó que había desaparecido y quiso reintegrarla contra el `main` actual (que había avanzado ~40 commits).
- **How to apply:** Si alguien pregunta dónde está el kiosk o por qué no está en main, la respuesta es: mergeada a develop, pendiente de subir a main vía PR.

## Decisiones técnicas no obvias

- **Ruta protegida por login** (`requireAuthGuard`), no es pública. Si en el futuro se quiere un kiosco que arranque sin sesión, hay que crear un guard específico o quitar el guard.
- **Hilton-branded hardcoded**: el layout carga `/assets/img/kiosk/hilton-logo.png` directo. No es dinámico por operador. Si se quiere multi-brand, hay que mover a `OperatorDataState.metadata()`.
- **Input oculto con foco intencional**: `kiosk-scanner` usa `<input aria-hidden="true" tabindex="-1" autofocus>` para capturar teclado del escáner físico. Genera un warning benigno en consola ("Blocked aria-hidden on an element because its descendant retained focus") — **no arreglar**, es el diseño.
- **Flujo:** debounce 300ms sobre valueChanges → `RequestCarService.requestCar(ticket)` → success/error screen → auto-reset a idle en 5s.
- **Error handling**: mapea código `BR-051-TICKET-ALREADY-REQUESTED` a mensaje específico + ícono `car-crash`. Cualquier otro error usa mensaje default + `alert`. Severidad `error` si status >= 500, `warning` en otros.

## Deuda técnica arrastrada del merge

- **`.claude/settings.local.json` trackeado por accidente**: se coló en el primer commit del kiosk (`c6ed3a5`) antes de que `.gitignore` lo excluyera. Sigue en el árbol de develop tras el merge. Limpiar con `git rm --cached .claude/settings.local.json` en una PR aparte.
