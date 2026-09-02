---
name: Feature — URL de recibo en Guest Page tras pago Stripe
description: Decisión y plan para exponer un link al recibo después de pagar con Stripe; el transactionReference de get-parameters ya es el receiptId
metadata:
  type: project
---
Arrancó el 2026-09-02. Objetivo: que el usuario que paga con Stripe en la guest page tenga una URL donde ver lo que pagó — hoy solo Square muestra recibo.

## Decisión tomada

El link apunta a **`https://receipts.corepark.com/{transactionReference}`** (nuestra app de recibos), y la app redirige a `pay.stripe.com` cuando `paymentOperator` es `STRIPE`. Un solo dominio para todos los gateways; Square queda como la excepción histórica.

Descartado: linkear directo al `receipt_url` de Stripe. Habría necesitado un endpoint nuevo, porque **Stripe.js no puede dar la URL** — `receipt_url` vive en el *Charge*, no en el *PaymentIntent*, y `confirmPayment` solo devuelve el intent.

## El front ya tiene el id

`StripeGetParameters` (`core/models/definitions/stripe.ts:10-18`) trae `paymentIntentId` y `transactionReference`. Se guardan en `StripeState.parameters` en el init (`stripe-payment.component.ts:74`), y la fila de `stripe_transaction` existe desde el `get-parameters` (hay filas con `status = requires_payment_method`), así que el id está disponible **antes** del cobro. Cero endpoints nuevos en el front.

**Cuál de los dos usar:** el endpoint `/payment/receipts/{id}` **solo acepta UUIDs** — un `pi_...` responde `INVALID-INPUTS-FIELDS`. Y la convención de Stripe del PR #40 de notifications es `pi_...` **o el uuid del ledger** (`payment_gateway_payment_uuid`), **nunca `stripe_transaction.uuid`**. Así que el link web tiene que llevar el uuid del ledger.

**Pendiente de confirmar (bloquea escribir el front):** que `transactionReference` sea el uuid del **ledger** y no el de `stripe_transaction` — coinciden en la mayoría de los cargos y divergen en algunos. Se verifica abriendo un ticket Stripe en dev, viendo `get-parameters` en el network tab, y comparándolo contra `payment_gateway_payment.payment_gateway_payment_uuid`.

## Trampas del lado guest page

- **Orden de `reset()`**: `StripeDialogComponent.onCloseDialog()` llama `StripeState.reset()`, que pone `parameters` en null. Hay que leer el `transactionReference` **antes** o el link se pierde.
- **`PaymentDetailState` nunca se limpia** y es `providedIn: 'root'`. Hoy no se nota porque solo Square lo escribe en el camino de éxito; en cuanto Stripe también lo escriba, un pago fallido tras uno exitoso mostraría el recibo anterior.
- `<square-payment-receipt>` tiene el copy y el nombre atados a Square pero el comportamiento es genérico — conviene generalizarlo en vez de duplicar.
- `Environment` no tiene campo para la URL de la app de recibos. En dev el host es una distribución CloudFront de las que lista el CORS de `application-dev.yml`, **sin identificar cuál**.
- `StripeDialogComponent.onDone()` hoy solo hace snackbar + cierra: no hay superficie donde viva el link. Lo consistente con Square es abrir `PaidTicketDialog` (que ya trae el gate `@if (isReceiptUrlConfigured())`).

## Cambios por repo

1. **`ms-payment-service`** (el único cambio de backend que falta; repo no clonado): resolver gateway 4 en `/receipts/{id}` espejeando `CommonServiceImpl.getStripeReceiptUrl` de notifications (por uuid del ledger), + agregar campo de URL al schema de `Receipt`, + distinguir "recibo no disponible todavía" para que el cliente reintente.
2. **`frontend-receipt`**: declarar y usar `paymentOperator`, redirigir en operadores hospedados, y arreglar el manejo de error (el endpoint responde 200 con `receipt: null`, así que hay que ramificar por `code !== 'OK'`).
3. **`frontend-guest-page`**: lo de arriba.
4. **`ms-valet-service`**: **ningún cambio.** Se intentó devolver `st.uuid` para gateway 4 y se revirtió — el fallback al uuid del ledger ya es el correcto.
5. **`ms-notifications-service`**: **ningún cambio** para Stripe, ya resuelto por el PR #40. Solo queda FreedomPay, que es otro alcance.

## Caso borde ya explicado

En dev, 148 cargos Stripe `succeeded` pero solo 132 con `receipt_url` — 16 sin (11%). **Causa confirmada:** el `receipt_url` lo escribe el webhook `charge.updated` de PaymentService, que puede tardar segundos respecto al cobro. Notifications ya lo modela con `ERR_BS_STRIPE_RECEIPT_NOT_AVAILABLE` para que el caller reintente; el front debe mostrar "recibo en proceso", no un error definitivo.
