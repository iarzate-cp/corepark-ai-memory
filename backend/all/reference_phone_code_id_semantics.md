---
name: phone_code_id — es countryId, no dial number
description: La columna phone_code_id en parking_service (y en general en el schema) referencia country_id de custom.cat_country; NO es el dial code (52, 1, etc.)
type: reference
---
## El problema

El naming es confuso. `phone_code_id` **NO** guarda el dial number (`52`, `1`, `34`, ...). Guarda un FK a `custom.cat_country.country_id`.

`custom.cat_country`:
- `country_id`   — PK
- `phone_code`   — el dial number (52, 1, ...)
- `country_code` — ISO (MX, US, ...)
- `name`         — nombre del país

## Prueba de campo

**Catalog SQL** (`ms-backoffice-service`, `CatalogueDaoImpl.java:34-49`):
```sql
SELECT country_id countryId, phone_code phoneCode, country_code countryCode
FROM custom.cat_country cc
```

**FK confirmado** — múltiples DAOs joinean así:
```sql
JOIN custom.cat_country cc ON cc.country_id = pl.phone_code_id      -- ms-oauth-service/CredentialsDaoImpl.java:149
LEFT JOIN custom.cat_country cc ON ps.phone_code_id = cc.country_id -- microservice-reports/TicketListReportDaoImpl.java:130
```

**Writer explícito** — `ms-valet-service/ValetDao.java:1373`:
```java
parameters.addValue("phoneCodeId", request.getGuest().getPhone().getCountryId());
```

Aquí el nombre del parámetro del bean es `countryId` pero se mapea a la columna `phone_code_id`. Zero ambigüedad: `phone_code_id` es el countryId.

## Trampa a evitar

El endpoint `POST /reports/ticket-list/get-ticket-detail` (en `microservice-reports/TicketListReportDaoImpl.java:86`) devuelve `guestPhoneCode` como el **dial number**, no como el countryId:

```sql
cc.phone_code guestPhoneCode
```

Si vas a prefill un mat-select de países mapeando por `countryId`, ese endpoint **no** sirve — el valor es el dial, y para el edit endpoint necesitas el countryId. Usa un endpoint dedicado o alinea el SQL para devolver `ps.phone_code_id`.

Ejemplo de endpoint que sí devuelve el countryId (para el edit del portal de validación):
- `GET /valet/validations/ticket-info/{ticket}?checkIn=...` (feature agregada 2026-08-06)

## Catálogo de países en ms-valet-service

Además del `/backoffice/catalogue/init` (bulk de todos los catálogos), el valet-service tiene su propio endpoint dedicado:

- **`GET /catalogs/country-codes`** (`ValetController.java:170`).
- Devuelve `{ countriesList: [{ countryId, countryName, countryIsoCode, phoneCode }] }`.
- **Cacheado in-memory** como Spring `@Bean` al startup del JVM (ver `com.corepark.valet.config.InitialCatalogLoading`); la query no corre por request.
- Es GET puro, sin body — el `operatorBodyInterceptor` de frontend-validation no lo toca.
- Servido desde el gateway como `/valet/catalogs/country-codes` (StripPrefix=1).
