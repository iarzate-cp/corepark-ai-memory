---
name: Flujo de recibos multi-gateway — GET /payment/receipts/{receiptId}
description: El receiptId es el uuid de la tabla de transacción del gateway; dónde vive el recibo de cada gateway y por qué el endpoint responde 200 en los errores
metadata:
  type: project
---
## Dónde vive

`GET /payment/receipts/{receiptId}` en **`ms-payment-service`**, clonado en `~/Dev/Back-End/ms-payment-service` desde el 2026-09-02. El gateway lo rutea con `Path=/payment/**` + `StripPrefix=1` y lo deja **público** (`ResourceServerConfig.java:101` → `GET /payment/receipts/** permitAll`). El spec de Scalar del Payment Service viene con `paths: []`, así que el MCP no sirve para esto — usar `api-specs.corepark.com`.

Front que lo consume: **`~/Dev/frontend-receipt`** (Angular 19, `receipts.corepark.com`, una sola ruta `:receiptId`).

## El receiptId es el uuid de la tabla del gateway

Verificado contra dev el 2026-09-02:

| id probado | resultado |
|---|---|
| `windcave_transaction.uuid` | `code: OK` + recibo |
| `payment_gateway_payment_uuid` del **mismo** cargo | `DB-DATA_NOT_FOUND` |
| `stripe_transaction.uuid` | `DB-DATA_NOT_FOUND` (el resolver solo consulta windcave) |

El endpoint **valida que el id sea un UUID**: un `pi_...` o `ch_...` responde `INVALID-INPUTS-FIELDS`.

**Ojo, Stripe es la excepción a la regla de arriba.** Su referencia de recibo NO es `stripe_transaction.uuid` sino el `pi_...` o el uuid del ledger (`payment_gateway_payment_uuid`) — así lo resolvió el PR #40 de notifications. Como el endpoint solo acepta UUIDs, en Stripe la referencia web tiene que ser la del ledger. Detalle en [[bug_sms_receipt_missing_gateways]].

`transactionReference` (Windcave y Stripe `get-parameters`) es el uuid que el caller genera al iniciar la transacción; para Windcave coincide con `windcave_transaction.uuid`. **Para Stripe falta confirmar** si es el uuid del ledger o el de `stripe_transaction` — los dos coinciden en la mayoría de los cargos y divergen en algunos.

`company.parking_service` **no tiene** columna de recibo (solo `uuid` y `lodging_guest_info_uuid`).

## El recibo del huésped es por ticket

`GET /payment/receipts/guest/{ticketUuid}` — **el que usa la Guest Page**, distinto del `/receipts/{receiptId}` de arriba. La llave es `company.parking_service.uuid`, y devuelve el ticket con **todos** sus cargos: desglose por pago (parking, tax, fee, tarifa nocturna, tip), refunds, cargos declinados, tarjeta, logo del operador y el `receiptUrl` hospedado de cada pago. Lo sirve `ReceiptServiceImpl.getGuestTicketReceipt`.

Este sí maneja bien los errores: **404 real** con `ERR-RECEIPT-001-TICKET-NOT-FOUND`, no el 200-con-null del endpoint viejo. Pero **solo cubre "el ticket no existe"**: un ticket válido sin cobros devuelve 200 con `payments: []` y `totalAmount: 0`, y el `@for` de `<guest-receipt>` no tiene bloque `@empty`, así que pinta el encabezado y nada más.

Front: `frontend-receipt` ruta `/ticket/:ticketUuid`. Lo consumen la Guest Page, `frontend-backoffice` (env `ticketReceiptsBaseUrl`) y valet por SMS en el checkout (`SmsService.getReceiptShortLink`, config `receipt-page.domain`).

## Recibo por gateway

Mapeo canónico en `microservice-reports` → `TicketListReportDaoImpl.QRY_PAYMENT_RECEIPTS` (~línea 1011). Catálogo `custom.cat_payment_gateway` (nombres en UPPERCASE): 1 SQUARE, 2 WINDCAVE, 3 FREEDOMPAY, 4 STRIPE, 5 SHIFT4.

| id | Gateway | De dónde sale |
|---|---|---|
| 1 | SQUARE | URL derivada: `squareup.com/receipt/preview/` + `pgp.payment_id` |
| 2 | WINDCAVE | Texto térmico: `windcave_hit_transaction.customer_receipt` + `_width` |
| 3 | FREEDOMPAY | Texto térmico: `freedompay_afcc_transaction.customer_receipt` + `_width` |
| 4 | STRIPE | URL guardada: `company.stripe_transaction.receipt_url` (join `st.payment_intent_id = pgp.payment_id`) |

Square y Stripe son páginas hospedadas (URL); Windcave y FreedomPay son texto de terminal que **nunca** se debe re-wrapear — se corta con el `width` con el que formateó el terminal.

## Trampa: el endpoint responde HTTP 200 en los errores

```
GET /payment/receipts/<uuid-inexistente>
HTTP 200  {"code":"DB-DATA_NOT_FOUND","message":"...","receipt":null}

GET /payment/receipts/not-a-uuid
HTTP 200  {"code":"INVALID-INPUTS-FIELDS","message":"...","receipt":null}
```

La respuesta OK trae `receipt: { customerReceipt, customerReceiptWidth, paymentOperator }` — **`paymentOperator` ya existe como discriminador** (ej. `"WINDCAVE"`), aunque `frontend-receipt` hoy no lo declara ni lo usa.

**Cualquier cliente debe ramificar por `code !== 'OK'`, no por status HTTP.** `frontend-receipt` no lo hace: su `receipt-error.component.ts` mapea 401/404/400/500 que nunca llegan, y `receipt-page.component.ts:38` explota sobre `receipt: null`.

## Consumidores del receiptId

- **`mobile_worker` (Android)** — el consumidor real. `BulkCheckOutValidateListFragment.kt:280` hace `receiptId = if (gateway == 1) paymentId else response.receiptId`, tomándolo del transaction-detail de valet. Su `receipt_journey/` manda el SMS.
- **`ms-valet-service`** lo expone en `TransactionsDao.java:59` (`QUERY_GET_TRANSACTION_DETAIL`): `COALESCE(CASE WHEN gateway=2 THEN wt.uuid END, pgp.payment_gateway_payment_uuid)`. **No lo "arregles" para que devuelva `stripe_transaction.uuid`** — el fallback al uuid del ledger es justo lo que el lookup de Stripe espera. Se intentó el 2026-09-02 y se revirtió.
- **Guest page** ya **no** usa este endpoint ni el `receiptUrl` de `pay-ticket`: linkea al recibo por ticket con `ticket.ticketUuid`. Ver [[feature_stripe_receipt_url]] en frontend/guest-page.

Ver [[bug_sms_receipt_missing_gateways]].
