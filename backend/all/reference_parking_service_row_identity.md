---
name: parking_service — identidad de fila (PK real incluye check_in)
description: El PK de company.parking_service es (operator, location, ticket, check_in); un mismo ticket string se repite con distintos check_ins
type: reference
---
## Constraint

Confirmado vía `pg_constraint` (2026-08-06):

```
pk_parking_service | PRIMARY KEY (operator_company_id, parking_location_id, check_in, ticket)
```

## Implicación

Un mismo `ticket` string (típicamente numeración diaria de valets — `1`, `2`, `3`, ...) se **reutiliza** a lo largo del tiempo. Ejemplo real en `coreparkdev`: `ticket='3'` tiene 19 filas para el mismo operator+location, con distintos check_ins entre 2020 y 2026.

Al escribir queries que identifiquen una fila específica (SELECT, UPDATE, DELETE), **siempre** incluir `check_in` en el WHERE. Si solo se filtra por `operator+location+ticket`, `queryForObject` truena con `IncorrectResultSizeDataAccessException: Incorrect result size: expected 1, actual N`.

## Formato del check_in

- En la BD: `TIMESTAMP WITH TIME ZONE` (guardado siempre en UTC internamente en Postgres).
- En las APIs de report (frontend consume): el JSON viene en **local time sin offset** (ej. `2021-10-08T15:41:17.924107`), no en UTC.
- Para hacer match desde el frontend cuando se recibe como String:

```sql
JOIN company.parking_location pl ON pl.operator_company_id = ps.operator_company_id
                                 AND pl.parking_location_id = ps.parking_location_id
WHERE ps.check_in = (:checkIn::timestamp WITHOUT TIME ZONE AT TIME ZONE pl.parking_location_tz)
```

Este patrón ya lo usan `TicketListReportDaoImpl` (microservice-reports) y `ValidationsDaoImpl.getTicketInfo/updateTicketInfo` (ms-valet-service, feature edit ticket info).

## Otras columnas relevantes

- `check_out` (nullable) — ticket sigue "activo" mientras sea NULL.
- `phone_code_id` (int) — FK a `custom.cat_country.country_id`. Ver `reference_phone_code_id_semantics.md`.
- Se toma como writer al `mobile_worker` (Android) para muchos campos; el backend edita/lee.
