---
name: project_trx_integration_status
description: "Estado de la integración TRX en la guest page al 2026-09-21 — qué está listo, qué espera a backend y qué a TRX"
metadata: 
  node_type: memory
  type: project
  originSessionId: a1632e92-0e98-42a8-bf7c-9a40c171e518
  modified: 2026-09-21T22:35:44.893Z
---

Al 2026-09-21 el flujo de pago con tarjeta de TRX está completo y mergeado a `feature/staging`, pero **nunca se ha ejecutado un cobro real** — ni un decline. Ese es el primer pendiente de verificación.

Bloqueos que **no** son nuestros:

- **Google Pay**: cableado y listo. La sesión de TRX llega con `"googlePay": false`, así que la librería borra el campo. Falta que backend mande `GooglePay: "1"` en el Initialize. **No depende del hostname** — eso es solo Apple Pay, y estaban agrupados en el plan, que es lo que lo frenaba.
- **Apple Pay**: la página y el botón están construidos, pero falta elegir y crear los hostnames (`trx-pay.corepark.com` / `dev-trx-pay.corepark.com`, ninguno resuelve), que TRX verifique el dominio bajo su merchant id de Apple, y el `ApplePay: "1"` de backend. Del lado nuestro tampoco está cableado el salto desde la guest page a ese host ni el retorno — `trxPayUrl` está vacío a propósito.
- **Custom tip**: el flujo existe y es compartido con Stripe. Si no se ve el campo, es `tipConfig.customAmountAllowed` en false para ese location — configuración de backoffice.

**Card on File se aparcó** por decisión de Israel: TRX tiene problemas con su documentación. Backend sí lo entregó (16-sep) pero no hay consumidor en el frontend; el andamiaje (`TrxCardOnFile`, `cardOnFile()`, sus tipos) quedó en el branch sin usar.

Asunciones sin confirmar con TRX: la URL del script en producción (hoy apunta al mismo host que dev), dónde va `CardSecurity` en un pago Apple Pay, y si `billingAddress.postal` (llega `required: true`) hace falta en una venta simple.

Ver [[reference_trx_paypage_library]] y [[reference_trx_integration_plan_doc]].
