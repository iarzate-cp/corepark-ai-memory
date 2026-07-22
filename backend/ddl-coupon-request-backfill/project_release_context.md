---
name: Coupon request CREATE backfill — release 2026-07-22
description: Data-only DDL that inserts missing 'CREATE' rows in company.coupon_request for orphan coupon batches, so they become visible again to the Back Office endpoints. Waiting on backend review before DEV run.
type: project
---

## Problema

`GET /v1/coupons` (y por consecuencia `PUT`, `DELETE` y `/history`) no muestra
cupones que están en `company.coupon` pero no tienen su fila `CREATE` en
`company.coupon_request`. El query `QRY_FIND_SUMMARIES_BY_LOCATION`
(`ms-backoffice-service/coupon/dao/impl/CouponDaoImpl.java:143-207`) usa
`JOIN LATERAL` INNER contra `coupon_request … action='CREATE'` — sin esa fila
el cupón se filtra silenciosamente.

## Alcance verificado en PROD (2026-07-22)

- **47 cupones huérfanos** distribuidos en **23 pares (operator, location)**
  y **8 operadores** (2, 19, 41, 98, 233, 243, 274, 283).
- El batch del Fairmont Long Beach (274, 3) son 14 cupones (ids 430-443,
  100 vouchers cada uno) — el caso reportado que arrancó la investigación.
- Los orphans son de dos categorías:
  - **Post-migration**: cupones creados después del release 2026-07-15
    mediante inserts manuales que olvidaron `coupon_request`.
  - **Pre-migration**: cupones que existían antes del release, cuando la
    tabla `coupon_request` no existía. El release nunca hizo backfill.
- Se validó que los 14 del batch de Fairmont tienen íntegros los otros
  joins (`coupon_type_time`, `partner`, `coupon_voucher = 100`), así que
  backfillear solo el `CREATE` es suficiente para hacerlos visibles.

## Decisión de attribution

`requested_by` se auto-resuelve al **OWNER del operador**
(`company.employee.profile_id = 1`). Se verificó que los 8 operadores
target tienen exactamente un owner cada uno:

| Operator | Owner id | Nombre / dominio |
|---|---|---|
| 2   | 1  | Jorge Perez (secureparkinginc) |
| 19  | 1  | Itzel Soria (supremeparking) |
| 41  | 1  | Tim Earlywine (streamlinevalet) |
| 98  | 3  | Arya Alexander (curbstand) |
| 233 | 25 | Jordan Razey (epicvalet) |
| 243 | 1  | Mubashar Allaudin (eminentvalet) |
| 274 | 1  | Eric Martin (parkpca) |
| 283 | 1  | Kia Shakoori (propark) |

## Decisiones que quedan documentadas en el DDL

- `created_at` = `coupon.created_at` (fidelidad histórica del audit, no
  `NOW()` del backfill).
- `vouchers_delta` = `COUNT(coupon_voucher)` por cupón — puede ser 0 si
  el batch está vacío (el CHECK constraint lo permite).
- Idempotente para `CREATE` (NOT EXISTS guard).
- Global — un solo INSERT ataca los 47 orphans.
- Rollback vía `RETURNING id` — el reviewer captura los IDs devueltos y
  hace `DELETE FROM coupon_request WHERE id IN (...)`.

## Estado

- ⏳ **En review por backend** antes del run en DEV (workflow estándar
  Corepark: dev primero, prod después; DEV y PROD comparten data).
- Bloqueador secundario: Israel no tiene permiso `ssm:StartSession`
  sobre el bastion PROD `i-06d62d082a87d3a38`. Solicitud IAM pendiente.

## Causa raíz pendiente

Cómo entran los cupones "manuales" post-migration sin `coupon_request`.
Si es un script/import/endpoint recurrente, hay que arreglar el origen
para que el problema no se regenere. Investigar después del pilot.
