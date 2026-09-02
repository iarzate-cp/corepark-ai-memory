---
name: SMS de recibo — Stripe RESUELTO (PR #40), FreedomPay sigue roto
description: PaymentService ya soporta STRIPE vía lookup en BD; FREEDOMPAY sigue ausente y lanza ERR_BS_PAYMENT_CONF_NOT_FOUND. Incluye la convención de identificador de recibo de Stripe
metadata:
  type: project
---
`POST /notifications/sms/send-receipt` arma el link con `PaymentService.getTicketUrl() + paymentId`, y el gateway se lee de BD (`CommonDAOImpl` → `custom.cat_payment_gateway.name`, UPPERCASE). `PaymentService.fromString` es **name-based**, así que un gateway ausente del enum devuelve null y `CommonServiceImpl.getPaymentService` lanza `ERR_BS_PAYMENT_CONF_NOT_FOUND`.

## Estado (verificado en `main` el 2026-09-02)

| Gateway en BD | Enum | Resultado |
|---|---|---|
| SQUARE | ✅ | `squareup.com/receipt/preview/` + paymentId |
| WINDCAVE | ✅ | `receipts.corepark.com/` + paymentId |
| STRIPE | ✅ | **resuelto por PR #40** (ver abajo) |
| **FREEDOMPAY** | **ausente** | **`ERR_BS_PAYMENT_CONF_NOT_FOUND`** |
| SHIFT4 | ✅ | link literal `"NOT DEFINED" + paymentId` |

FreedomPay guarda texto térmico (`freedompay_afcc_transaction.customer_receipt`), así que su fix es `FREEDOMPAY("https://receipts.corepark.com/")` — pero ojo, eso depende de que el resolver de `ms-payment-service` sepa atenderlo.

## Stripe: convención de identificador (PR #40, `fix/stripe-receipt-lookup-by-transaction-uuid`)

Stripe **no concatena base + id** — su recibo es una URL completa por cargo, persistida por el webhook `charge.updated` en `company.stripe_transaction.receipt_url`. Por eso el enum declara `STRIPE(null)` y `SmsServiceImpl.sendSMSReceipt` ramifica a `commonService.getStripeReceiptUrl(...)`.

Ese lookup (`CommonServiceImpl.getStripeReceiptUrl`) acepta **dos** referencias y decide parseando:

- **si el `paymentId` parsea como UUID** → resuelve por `pgp.payment_gateway_payment_uuid` (el uuid del ledger), haciendo join `st.payment_intent_id = pgp.payment_id`
- **si no** → resuelve por `st.payment_intent_id` directo (un `pi_...`)

**⚠️ La referencia de recibo de Stripe NO es `stripe_transaction.uuid`.** Es el `pi_...` o el uuid del ledger (`payment_gateway_payment_uuid`). Los dos uuids **coinciden en la mayoría de los cargos y divergen en algunos** (caso real en dev: ledger `4c9872bf-…` vs `st.uuid` `b5b45095-…`), así que usar el equivocado falla de forma intermitente. `ms-valet-service` ya devuelve el correcto en `TransactionsDao` — su fallback a `payment_gateway_payment_uuid` **es lo correcto, no un bug**.

Cuando la URL no está todavía (el webhook puede tardar segundos) o no hay fila, ambos casos salen como `ERR_BS_STRIPE_RECEIPT_NOT_AVAILABLE` para que el caller reintente. Eso explica las filas `succeeded` sin `receipt_url` (132 de 148 en dev).

## Inconsistencia que queda

`ReceiptSMS.receiptType` es `@NotNull` pero **nunca se lee** — el gateway se decide por BD. `ReceiptType` (ya tiene SQUARE/WINDCAVE/FREEDOMPAY/STRIPE) y `PaymentService` (le falta FREEDOMPAY) son dos taxonomías paralelas; el `CLAUDE.md` del repo advierte explícitamente que hay que actualizar **ambas** cuando entra un gateway.

`frontend-valet-web` manda el body equivocado (`{ticket, phoneNumber, phoneId}` en `tickets-service.ts:184-198`); el contrato correcto es el del Android (`ReceiptViewModel.kt:171-175`).

Ver [[project_receipt_flow_multi_gateway]].
