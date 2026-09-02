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

`StripeGetParameters` (`core/models/definitions/stripe.ts:10-18`) trae `transactionReference`, que es el uuid de la transacción del gateway = el receiptId (ver [[project_receipt_flow_multi_gateway]] en backend/all). Se guarda en `StripeState.parameters` en el init (`stripe-payment.component.ts:74`), y la fila de `stripe_transaction` existe desde el `get-parameters` (hay filas con `status = requires_payment_method`), así que el uuid está disponible **antes** del cobro. Cero endpoints nuevos en el front.

**Pendiente de confirmar:** que el `transactionReference` de Stripe sea literalmente `stripe_transaction.uuid`. Probado por simetría en el lado Windcave, no directo en Stripe. Se verifica abriendo un ticket Stripe en dev, viendo `get-parameters` en el network tab, y comparando contra la fila.

## Trampas del lado guest page

- **Orden de `reset()`**: `StripeDialogComponent.onCloseDialog()` llama `StripeState.reset()`, que pone `parameters` en null. Hay que leer el `transactionReference` **antes** o el link se pierde.
- **`PaymentDetailState` nunca se limpia** y es `providedIn: 'root'`. Hoy no se nota porque solo Square lo escribe en el camino de éxito; en cuanto Stripe también lo escriba, un pago fallido tras uno exitoso mostraría el recibo anterior.
- `<square-payment-receipt>` tiene el copy y el nombre atados a Square pero el comportamiento es genérico — conviene generalizarlo en vez de duplicar.
- `Environment` no tiene campo para la URL de la app de recibos. En dev el host es una distribución CloudFront de las que lista el CORS de `application-dev.yml`, **sin identificar cuál**.
- `StripeDialogComponent.onDone()` hoy solo hace snackbar + cierra: no hay superficie donde viva el link. Lo consistente con Square es abrir `PaidTicketDialog` (que ya trae el gate `@if (isReceiptUrlConfigured())`).

## Cambios por repo

1. **`ms-payment-service`** (bloqueante, repo no clonado): resolver gateway 4 por `stripe_transaction.uuid` + agregar campo de URL al schema de `Receipt`.
2. **`frontend-receipt`**: declarar y usar `paymentOperator`, redirigir en operadores hospedados, y arreglar el manejo de error (el endpoint responde 200 con `receipt: null`).
3. **`frontend-guest-page`**: lo de arriba.
4. **`ms-valet-service`** rama `feature/stripe-receipt-id` (creada desde `main` el 2026-09-02): devolver `st.uuid` para gateway 4 en `TransactionsDao.java:59`. El fallback actual coincide con el uuid correcto en 4 de 5 cargos de dev pero no en todos → bug intermitente.
5. **`ms-notifications-service`**: ver [[bug_sms_receipt_missing_gateways]].

## Caso borde a resolver

En dev, 148 cargos Stripe `succeeded` pero solo 132 con `receipt_url` — **16 cobros exitosos sin recibo (11%)**. Hay que definir el estado "recibo no disponible" y averiguar si es una carrera con el webhook que puebla la columna.
