---
name: SMS de recibo truena en Stripe y FreedomPay (enum PaymentService incompleto)
description: PaymentService en ms-notifications-service solo tiene WINDCAVE/SQUARE/SHIFT4; con Stripe o FreedomPay fromString devuelve null y lanza ERR_BS_PAYMENT_CONF_NOT_FOUND
metadata:
  type: project
---
`POST /notifications/sms/send-receipt` arma el link así (`SmsServiceImpl.java:460-461`):

```java
final PaymentService paymentService = commonService.getPaymentService(operatorCompanyId, parkingLocationId);
final String finalMessage = smsBody + " " + paymentService.getTicketUrl() + request.getPaymentId();
```

El gateway se lee de BD (`CommonDAOImpl.java:36-48` → `custom.cat_payment_gateway.name`), y el enum `notifications/enums/PaymentService.java` solo declara `WINDCAVE`, `SQUARE` y `SHIFT4`. Los nombres en BD son UPPERCASE exactos, así que:

| Gateway en BD | `PaymentService.fromString` | Resultado |
|---|---|---|
| SQUARE | ✅ | link a `squareup.com/receipt/preview/` |
| WINDCAVE | ✅ | link a `receipts.corepark.com/` |
| **STRIPE** | **null** | **`ServiceException(ERR_BS_PAYMENT_CONF_NOT_FOUND)`** en `CommonServiceImpl:33-37` |
| **FREEDOMPAY** | **null** | **misma excepción** |
| SHIFT4 | ✅ | link literal `"NOT DEFINED" + paymentId` |

Ambos gateways que faltan deben apuntar a `https://receipts.corepark.com/` — Stripe porque el resolver le devolverá la URL hospedada, FreedomPay porque también guarda texto térmico. Ver [[project_receipt_flow_multi_gateway]].

## Dos inconsistencias alrededor

- **`ReceiptSMS.receiptType` es `@NotNull` pero nunca se lee.** El enum `ReceiptType` (SQUARE/WINDCAVE/FREEDOMPAY) y `PaymentService` (WINDCAVE/SQUARE/SHIFT4) modelan lo mismo y están desalineados. Si se toca esto, unificarlos evita agregar cada gateway nuevo en dos lugares con criterios distintos.
- **`frontend-valet-web` manda el body equivocado**: `{ticket, phoneNumber, phoneId}` (`tickets-service.ts:184-198`) cuando el bean espera `{paymentId, userPhoneNumber, receiptType}`. El contrato correcto es el del Android (`ReceiptViewModel.kt:171-175`). El botón de la web debería estar dando 400 — sin confirmar en runtime.
