---
name: Audit trails should not gate operational visibility
description: When designing summary/read queries in Corepark microservices, treat audit metadata (who/when created/edited/downloaded) as enrichment, not as a hard prerequisite. Missing audit rows should not make the underlying operational record disappear.
type: feedback
---

Cuando el query de una lectura operativa (listado / detalle / edición /
borrado) trae información de una tabla de auditoría (`*_request`, `*_download`,
`*_history`, o similares), esa hidratación debe ser vía `LEFT JOIN` — nunca
`INNER JOIN` ni `JOIN LATERAL` sin salida.

**Regla:** el registro operativo (`coupon`, `parking_service`, `charger`,
etc.) es la fuente de verdad de "esto existe y se puede usar". Que le falte
un evento de audit no lo debe invisibilizar ni bloquear operaciones sobre él.

**Why:** El principio salió explícito en el review del backend de
`ms-backoffice-service` el 2026-07-22 sobre `GET /v1/coupons`. Cupones
insertados manualmente en `company.coupon` sin fila `CREATE` en
`company.coupon_request` eran filtrados silenciosamente por un
`JOIN LATERAL` INNER en `QRY_FIND_SUMMARIES_BY_LOCATION`. La reacción del
reviewer fue arquitectural, no anecdótica: *"la bitácora es tracing vs
operativa; no debe romper el pull de cupones"*. La generalización aplica a
cualquier microservicio con este patrón (payments/audit_log,
notifications/delivery_log, etc.).

**How to apply:**

1. **Al escribir un query nuevo**: si haces JOIN a una tabla de audit, es
   `LEFT JOIN` por defecto. Solo justifica `INNER` si el registro operativo
   sin audit es *definitivamente* inválido (raro).
2. **Al revisar código existente**: cualquier `JOIN LATERAL … audit` con
   `LIMIT 1` sin `LEFT` es un smell. Verifica que el registro sobrevive si
   la audit está ausente.
3. **Bean nullability**: los campos derivados de audit (`requestedBy`,
   `lastEventAt`, etc.) tienen que tipar como nullables (`Integer`, no
   `int`; `String`, no primitivos). Actualiza comentarios del bean.
4. **Reparación de datos**: cuando aparezcan orphans (data existente sin
   audit), el patrón preferido es **atribución vía UI existente** en el
   próximo edit — no un endpoint nuevo tipo `/attribute` ni un DDL de
   backfill que fabrique atribución sintética (ej. asignar al OWNER del
   operador). Ver ejemplo concreto en `ms-backoffice-service` `updateCoupon`
   / `deleteCoupon` (2026-07-22): el service inserta el `CREATE` faltante
   con el `requestedBy` del payload la primera vez que el usuario toca el
   registro.
5. **UI**: renderiza los campos audit condicionalmente. Un mensaje del tipo
   "Not attributed yet — pick an employee on the next edit" es mejor que
   `undefined` en el DOM y que un dato sintético que oculta el gap real.
