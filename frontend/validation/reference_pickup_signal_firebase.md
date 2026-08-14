---
name: se-al-de-pickup-vive-en-requestnotification-no-en-ticket-status
description: El flujo pickUp de ms-valet-service NUNCA escribe READY FOR PICKUP al campo status del ticket en Firebase; la única señal es requestNotification.status === PICKUP. Incluye la traza completa por los 2 microservicios y por qué el enum de TS tenía una entrada muerta.
metadata: 
  node_type: memory
  type: reference
  originSessionId: c4a21c78-ea14-4647-84b1-e960c4936548
  modified: 2026-08-14T01:23:21.374Z
---

Verificado 2026-08-13 leyendo el código de `Back-End/` y contra payloads reales
de Firebase (ticket 20197, ambiente dev).

## El hecho

Un ticket listo para entrega **conserva su `status` operativo** (`ATTEND`,
`EN ROUTE`, `PARK`…). El único rastro del pickup en Firebase es:

```
requestNotification.status === 'PICKUP'
```

`READY FOR PICKUP` **jamás aparece** en el campo `status` del nodo del ticket, ni
en `ixParkingLocationTicketStatus`. Filtrar por él en el frontend es letra muerta.

## Por qué — traza de escritura

`ValetService.pickUp()` (`ms-valet-service`) hace dos cosas que parecen una sola:

1. `valetDao.changeTicketStatus(request, ParkingServiceStatus.READY_FOR_PICKUP, checkIn)`
   → `ValetDao.changeTicketStatus()` es **JDBC puro** (`namedParameterJdbcTemplate.update`).
   Escribe a la BD relacional y **no toca Firebase**. Por eso el status 20 sí existe
   en el enum canónico y en reportes, pero no en el feed del board.
2. `firebaseFeignService.processSMSNotificationViaHttp(...)` con un `SMSNotification`
   donde **nunca se llama `setTicketStatus(...)`** → el campo viaja `null`.
   (Contraste: `TicketsService:218` y `:323` sí lo setean, para `REQUEST` y `CANCEL`.)
3. En `ms-firebase-service`, `NotificationServiceImpl.updateTicketRequestNotification()`
   guarda el status del ticket detrás de un `if (request.getTicketStatus() != null)`.
   Como llega `null`, se salta y solo escribe el subnodo `requestNotification/*`.

**Why:** el nombre del método (`changeTicketStatus`) y el del enum sugieren que la
transición se propaga al feed. No lo hace: BD y Firebase son dos caminos separados,
y el de Firebase depende de que alguien setee `ticketStatus` explícitamente.

## PICKUP es sticky

Una vez escrito, `requestNotification.status` se queda en `PICKUP` mientras
`ticket.status` sigue avanzando. Mismo ticket 20197, dos momentos:

| `status` | `requestNotification.status` | `notifiedAtEpoch` |
| --- | --- | --- |
| `ATTEND` | `PICKUP` | 1786667356 |
| `EN ROUTE` | `PICKUP` | 1786667824 |

Ninguno de los dos trae `spotAssignment` / `row` / `spot`.

**How to apply:** para saber si un ticket está listo para el guest, usar siempre
`requestNotification?.status === RequestNotificationStatus.Pickup` — hay un util
compartido en `core/utils/ticket-pickup.ts`. Nunca compararlo contra `TicketStatus`.
La entrada `ReadyForPickup = 'READY FOR PICKUP'` se **eliminó** de
`core/enums/ticket.ts` el 2026-08-13 justo por esto. Si vuelve a aparecer en el
enum, es señal de que alguien repitió el error. Ver [[project_vms_kanban_redesign]]
y [[reference_parking_service_status_enum]].
