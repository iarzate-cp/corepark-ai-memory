---
name: project_trx_integration_status
description: "Estado de la integración TRX en la guest page al 2026-09-24 — qué está listo, qué espera a backend y qué falta para prod"
metadata: 
  node_type: memory
  type: project
  originSessionId: a1632e92-0e98-42a8-bf7c-9a40c171e518
  modified: 2026-09-24T16:48:22.803Z
---

Al 2026-09-21 el flujo de pago con tarjeta de TRX está completo y mergeado a `feature/staging`, pero **nunca se ha ejecutado un cobro real** — ni un decline. Ese es el primer pendiente de verificación.

Estado de los wallets y otros pendientes:

- **Google Pay**: ya aparece en el diálogo (visto en captura el 2026-09-23), así que backend ya manda `GooglePay: "1"`. **No se pide en navegadores con Apple Pay** (`isApplePayBrowser()` = `'ApplePaySession' in window`, decisión de Israel: Safari → Apple, resto → Google; incluye Chrome iOS). El botón va fuera de la caja de tarjeta, justo arriba de PAY.
- **Apple Pay** (2026-09-24, commit `3c923eb`, mergeado y pusheado a `feature/staging` en `2f51fe2`): se abandonaron los hostnames dedicados y la página `/trx-pay` (borrados junto con `trxPayUrl`). Ahora el botón de TRX se monta **dentro del diálogo**, en el mismo lugar que Google Pay, vía `trxWalletOf` (`core/utils/trx-wallet.ts`, enum `TrxWallet`) y el flag `environment.trxApplePayVerified` — **dev `true`, prod y local `false`**. Si el wallet falla al arrancar, las tarjetas siguen funcionando.
  - **Siguiente paso, decisión de Israel: probar primero en DEV.** Backend ya manda `ApplePay: "1"` (el query de sesión TRX del 2026-09-24 devuelve `applePay: true` y `googlePay: true`), así que ya se puede probar en Safari sobre dev.
  - **Para prod** basta con poner `trxApplePayVerified: true` en `environment.prod.ts` cuando pase DEV. No hay bloqueo de dominio.

**Archivos de asociación de Apple** (verificado con curl el 2026-09-24): conviven dos en `/.well-known/`, uno por proveedor, y **no chocan** porque Apple (TRX) lee el `.txt` y Square el que no tiene extensión:
- `apple-developer-merchantid-domain-association` (sin extensión, 9115 B, pspId `162188353C…`) → **Square**. Está en el repo (`src/assets/.well-known/`) y es idéntico en dev y prod.
- `apple-developer-merchantid-domain-association.txt` → **TRX**, distinto por dominio (dev 5763 B, prod 5750 B). **Jorge los subió a mano al hosting; no están en el repo.** Riesgo: si el deploy borra lo que no viene en el build, se pierden. Israel no ha decidido aún si versionarlos (requeriría assets por configuración en `angular.json`, porque dev y prod difieren).
- TRX verificó dev (`dgnw8qm5ijo58.cloudfront.net`): **vence el 10-mar-2027**. Prod es `guest.corepark.com`.

- **Custom tip**: el flujo existe y es compartido con Stripe. Si no se ve el campo, es `tipConfig.customAmountAllowed` en false para ese location — configuración de backoffice.

**Card on File se aparcó** por decisión de Israel: TRX tiene problemas con su documentación. Backend sí lo entregó (16-sep) pero no hay consumidor en el frontend; el andamiaje (`TrxCardOnFile`, `cardOnFile()`, sus tipos) quedó en el branch sin usar.

`ng test` no arranca (2026-09-23): falta `@socket.io/component-emitter` en node_modules, así que `trx-wallet.spec.ts` no se ha corrido. `ng build --configuration development` sí compila.

Asunciones sin confirmar con TRX: la URL del script en producción (hoy apunta al mismo host que dev), dónde va `CardSecurity` en un pago Apple Pay, y si `billingAddress.postal` (llega `required: true`) hace falta en una venta simple.

Ver [[reference_trx_paypage_library]] y [[reference_trx_integration_plan_doc]].
