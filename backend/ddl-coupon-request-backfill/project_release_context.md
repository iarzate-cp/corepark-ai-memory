---
name: Coupon request CREATE backfill — release 2026-07-22 (OBSOLETE)
description: Data-only DDL originally drafted to insert missing 'CREATE' rows for orphan coupon batches. Backend review rejected the premise ("audit trail should not gate operational visibility"); pivot was to a code change instead. DDL file kept for historical reference but should not be executed.
type: project
---

## Estado

**OBSOLETO.** Este DDL nunca se ejecutó ni en DEV ni en PROD. La solución final
fue un cambio de código en `ms-backoffice-service` + `frontend-commerce` que
shippeó a `feature/staging` el 2026-07-22.

## Contexto original

`GET /v1/coupons` (y por consecuencia `PUT`, `DELETE`, `/history`) no mostraba
cupones que estaban en `company.coupon` pero no tenían fila `CREATE` en
`company.coupon_request`. Los queries `QRY_FIND_SUMMARIES_BY_LOCATION` y
`QRY_FIND_SUMMARY_BY_ID` usaban `JOIN LATERAL` INNER contra la tabla de audit
— sin la fila `CREATE` el cupón se filtraba silenciosamente.

Alcance verificado en PROD (2026-07-22): **47 cupones huérfanos** distribuidos
en **23 pares (operator, location)** y **8 operadores**. Dos categorías:

- **Pre-migration**: cupones creados antes del release 2026-07-15 (cuando la
  tabla `coupon_request` aún no existía). Ese release nunca hizo backfill.
- **Post-migration**: cupones insertados a mano (imports de Excel, scripts
  ad-hoc) que no siguieron el flujo del service.

## Diseño original (rechazado)

Backfillear una fila `CREATE` por cupón huérfano, atribuida al OWNER del
operador (`company.employee.profile_id = 1` — verificado que los 8 operadores
target tenían exactamente un owner). El DDL vive en
`~/Dev/Back-End/ddl/DDL_2026_07_22_COUPON_REQUEST_BACKFILL/01_BACKFILL_...`
por si sirve de referencia, pero no debe correrse.

## Feedback de backend que forzó el pivot

> "La bitácora es un proceso de tracing vs un proceso de operativa. No me
> cuadra que la bitácora rompa el pull de los cupones."

Punto arquitectural válido: acoplar la visibilidad operativa a la
completitud del audit trail mezcla dos preocupaciones que deberían estar
separadas. Ver feedback general en Back-End/memory/.

## Diseño adoptado

Idea de Israel: **reutilizar el flujo de Edit existente en el frontend**.
No endpoint nuevo, no botón nuevo. Cambios:

- **BE**: `JOIN LATERAL` → `LEFT JOIN LATERAL` en las dos summary queries.
  `CouponSummary.requestedBy` y `lastEventBy` a `Integer` (nullables).
  `updateCoupon` y `deleteCoupon` detectan orphan (`current.getRequestedBy()
  == null`) e insertan la fila `CREATE` faltante con el `requestedBy` del
  payload antes de escribir el `EDIT` / `DELETE`.
- **FE**: audit fields de `CouponSummary` a `| null`. Template con `@if`
  guard mostrando "Not attributed yet — pick an employee on the next edit"
  para orphans. `lastEventLabel(null) → 'Unassigned'` como fallback.

## Ship

- BE commit `af56b02` en `feature/coupons`, mergeado a `feature/staging`
  (`d068155`), pusheado.
- FE commits `b1a910c` (env local) + `e786de7` (bump corepark-ui a 0.0.25)
  + `cfb207e` (fix de nullability) en `feature/coupons`, mergeados a
  `feature/staging` (`e224832`), pusheados.

## Causa raíz pendiente

Cómo entran cupones sin `coupon_request` en primer lugar. Puede ser un
trigger de compensación, validación en service layer que se saltó, o un
script/import ad-hoc. Fuera del scope de este release — backlog.
