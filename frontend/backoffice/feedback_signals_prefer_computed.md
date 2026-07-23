---
name: Prefer computed() over getter methods for state-derived helpers
description: In Angular signals, values derived from other signals should be computed(), not private methods that call signals internally
type: feedback
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
Cuando un valor se deriva de otros signals, usar `computed()`, no un método privado que llama signals adentro.

**Why:** El usuario me corrigió durante Activity by Rate Class: escribí `#locationName(): string { return this.#catalogueState.parkingLot()?.parkingLotName ?? '' }` como método privado. Debía ser `readonly #locationName = computed(() => this.#catalogueState.parkingLot()?.parkingLotName ?? '')`.

Los métodos re-evalúan cada llamada (waste). Los computeds cachean y sólo re-computan cuando cambia una dep. Además, un computed puede ser dep de otro computed, un método no.

**How to apply:**
- Si la función lee signals y retorna un valor derivado (sin side effects): **computed**.
- Si la función tiene side effects (`show()` de notificación, `.set()` de otro signal, HTTP call, etc.): **método** normal.
- Helpers puros que reciben parámetros y no leen state pueden quedarse como métodos privados o consts a nivel módulo — no son "state-derived".

**Excepción a tener en cuenta:** `viewChild.required(...)` no se puede declarar con `#private` (Angular lo rechaza con NG1053 — reflexión no soporta ES private). Usar `private readonly` para viewChild, aunque CLAUDE.md prefiera `#private` en general. Precedente: `dashboard-trend.component.ts:252`.
