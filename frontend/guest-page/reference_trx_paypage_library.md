---
name: reference_trx_paypage_library
description: "Dónde vive la librería PayPage de TRX y cómo averiguar su API real, porque la guía de integración no la documenta"
metadata: 
  node_type: memory
  type: reference
  originSessionId: a1632e92-0e98-42a8-bf7c-9a40c171e518
  modified: 2026-09-21T22:35:57.692Z
---

La *TRX Integration Guide v1.0* **no documenta el lado del navegador**. La API real se lee del bundle que TRX sirve:

- Librería: `https://paypage-fields.trxservices.net/bundle.js` (paquete `pay-page-js`, versión 1.0.3). Las URLs que estaban en los environments (`pay-page.sandbox.trxservices.net`, `pay-page.trxservices.net`) **no existen** — NXDOMAIN.
- Tool público de pruebas que TRX compartió con Israel: `https://pay-page-tool.trxservices.net` — sin cuenta, solo testing. Su `main.*.js` es la integración de referencia.
- App de los iframes: `https://paypage-fields.trxservices.net/static/js/main.*.js` — ahí vive el payload que devuelve el cifrado.

Cuatro cosas que la guía no dice y costaron una tarde el 2026-09-21:

1. Los **nombres de campo son kebab-case** (`card-number`, `expiration-date`, `cvv`, `google-pay`, `apple-pay`). El tool los escribe en camelCase pero los traduce con un mapa antes de llamar. Un nombre desconocido se descarta **en silencio**.
2. `PayPageJS.initialize` es **estática y devuelve la instancia**; los métodos van sobre ella. El que lee la tarjeta es `getEncryptedCardData()`, y el resultado viene anidado: `{ isSuccess, Encrypted: { Data, KeyId } }`.
3. `onCardTypeChange` es **obligatorio** aunque no lo uses; sin él invalida toda la configuración.
4. Los errores de configuración se acumulan en `session.errors` **en vez de lanzarse**, así que una integración rota parece sana con la pantalla vacía. Tipos: `wrong-config`, `wrong-field`, `container-error` (fatales) e `iframe-error`, `styles-error`, `permissions-error` (conviven con un formulario que funciona).

Los wallets no pasan por `getEncryptedCardData()`: aprueban en la hoja de TRX y el resultado llega por `onPaymentResult`, que es obligatorio en esos campos.

Ver [[project_trx_integration_status]].
