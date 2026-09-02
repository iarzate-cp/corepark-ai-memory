---
name: Feature — link al recibo del huésped en la Guest Page (entregado)
description: Botón en la página del ticket que abre receipts.corepark.com/ticket/{parking_service.uuid}; sirve para los 4 gateways; entregado y verificado en dev el 2026-09-02
metadata:
  type: project
---
Entregado y verificado end-to-end en dev el **2026-09-02**. Nació como "que el huésped que paga con Stripe pueda ver su recibo" y terminó siendo gateway-agnóstico.

## Contrato final

```
ticket-info  →  ticket.ticketUuid   (= company.parking_service.uuid, siempre presente)
                ↓
GP muestra el botón si  payments?.length > 0
                ↓
{ticketReceiptsBaseUrl}{ticketUuid}  →  receipts.corepark.com/ticket/{uuid}
                ↓
GET /payment/receipts/guest/{ticketUuid}  →  recibo por ticket con TODOS sus cargos
```

El recibo es **por ticket, no por cargo**: devuelve nombre, cuarto de hotel, la lista completa de pagos con desglose (parking, tax, fee, tarifa nocturna, tip), refunds, cargos declinados, la tarjeta, el logo del operador, y el `receiptUrl` hospedado del gateway por cada pago. Un ticket Card-on-File nocturno real trae 11+ cargos. Por eso no hace falta ningún identificador de cargo — ni `paymentIntentId`, ni `transactionReference`, ni `stripe_transaction.uuid`.

`ticketReceiptsBaseUrl` en `Environment`: dev y local `https://d1flyzjaekpl73.cloudfront.net/ticket/`, prod `https://receipts.corepark.com/ticket/`. Misma llave y mismos valores que `frontend-backoffice`, que ya linkeaba a esa página.

## Cómo llegó a este contrato (para no repetir el camino)

Primero expuse un nodo `receipt: { uuid } | null` donde el backend decidía si había recibo. En review, Jorge señaló que **ese uuid es el del ticket, no el de un recibo** — nombrar un bean por quien lo consume en vez de por lo que es. Tenía razón: se cambió por `ps.uuid ticketUuid` en `QUERY_GET_TICKET_INFO`, mapeado por el `BeanPropertyRowMapper` que ya existía. **+4 / −67 líneas.** Ver [[project_receipt_flow_multi_gateway]] en backend/all.

El costo fue perder el gate del backend, porque el uuid existe desde que nace el ticket. Se recuperó en el front con `payments?.length`, que ya viene en el payload con la misma condición (`payment_method IN ('C','B','P')`).

**Caso de borde conocido y aceptado:** un ticket totalmente reembolsado tiene `payments` vacío (la query excluye reembolsados con `AND psr.ticket_charge_uuid IS NULL`) pero la app de recibos sí renderiza refunds. A ese huésped se le esconde el botón de un recibo que sí existe. Si algún día importa, se arregla con un `ticket.hasReceipt` en el backend, no volviendo al bean.

## Dónde vive el link

En la **página del ticket**, entre el bloque de acciones y el QR, **fuera** de la bifurcación de `temporalCheckout` — un huésped en "See you soon!" ya pagó y es el más probable de querer su recibo. Sigue apareciendo también en `PaidTicketDialog` y `CarDeliveredDialog`. No vive en el branch de `showUnlockGate()`, que reemplaza toda la página.

Por qué no basta con los diálogos: en las locations con `allowRequest` el flujo post-pago muestra un snackbar y nunca abre diálogo, así que el link jamás se veía. Eso ya pasaba con Square antes.

El componente es `<ticket-receipt-link>` (heredado de `square-payment-receipt` vía `git mv`). Se eliminaron `PaymentDetailState` y el consumo del `receiptUrl` que devolvía `pay-ticket`: un solo link para los cuatro gateways.

## La carrera contra el webhook

Stripe cobra **dentro de un diálogo en la página del ticket**, así que nada re-instancia la página al terminar — a diferencia de Square, que navega desde `/payment`. Y el cargo llega a nuestro ledger por el webhook `payment_intent.succeeded`, segundos después de que el browser confirma (`InitStripeTransactionSubmitter.submit()` es quien escribe la fila).

Solución: `TicketInfoState.paymentRefreshRequests` como disparador, y la página hace polling —4 intentos cada 2s— **descartando los payloads intermedios**. Solo compromete storage y state cuando el payload trae **un pago más** de los que la pantalla ya conocía; contar en vez de preguntar "¿hay algún pago?" es lo que lo hace correcto en el segundo cobro de un ticket, no solo en el primero. Si el webhook no llega en la ventana, la pantalla conserva lo que tiene y el link aparece en la siguiente visita.

**Sin probar todavía:** ese refresh. Todos los tickets Stripe de dev son Overnight con Card-on-File y no muestran botón de pagar, así que nunca se ejercita `onDone()`. Hace falta un ticket con rate normal.

## Pendiente en otro repo

`<guest-receipt>` de `frontend-receipt` **no tiene bloque `@empty`** en su `@for` de pagos. Un ticket válido sin cobros devuelve 200 con listas vacías (el 404 solo cubre ticket inexistente), así que pintaría el encabezado y el cuerpo en blanco. Hoy el gate de `payments` lo evita desde la GP, pero la app debería manejar su propio estado vacío.
