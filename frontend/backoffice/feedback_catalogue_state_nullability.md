---
name: CatalogueState.init() puede ser null en primer render
description: Cualquier pipe/computed que consuma catalogueState.init() debe guardar contra null antes de destructurar; el signal arranca en null hasta que carga el catálogo
type: feedback
---
`CatalogueState.init` es un `signal<InitCatalogue | null>(null)` que se pobla asincrónicamente al arrancar la app. Los consumers deben chequear null antes de destructurar (`countries`, `paymentMethods`, etc.).

**Why:** En julio 2026, `CodePhoneNumberPipe` reventó al renderizar `TicketLogTableComponent` porque hacía `const { countries } = this.#catalogueState.init()` sin guarda — cuando la tabla se pintaba antes de que llegara la respuesta del catálogo, Angular tiraba `TypeError: Cannot destructure property 'countries' of ... as it is null`. Se arregló en `core/pipes/code-phone-number.ts` con `if (!init) return value`.

**How to apply:**
- Al escribir un pipe/computed/servicio que lee `catalogueState.init()`, hacer siempre `const init = this.#catalogueState.init(); if (!init) return <fallback>;` antes de destructurar.
- El fallback razonable en pipes es devolver el `value` crudo (no `''`) para no parpadear a vacío mientras carga — cuando el signal se pobla, Angular re-ejecuta el pipe.
- Si aparece un error `Cannot destructure property 'X' of 'this[#catalogueState].init(...)'`, la causa es siempre esta: falta la guarda.
- Aplica igual a otros signals nullable del proyecto (ej. cualquier `signal<T | null>(null)` que se pueble asincrónico).
