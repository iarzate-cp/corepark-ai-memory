---
name: Edit Ticket Info — feature del portal de validación
description: Nuevos endpoints GET+PUT en ms-valet-service para leer/editar guest y vehicle info; dialog nueva en overnight-report con cp-menu, mat-autocomplete de países y refresh automático
type: project
---
## Alcance

En el overnight-report (frontend-validation), agregar la capacidad de editar la info del guest (firstName, lastName, phoneCodeId, phoneNumber) y vehicle (maker, model, color, plate) de un ticket. Fecha implementación: 2026-08-06.

## Endpoints backend (ms-valet-service)

```
GET  /valet/validations/ticket-info/{ticket}?checkIn={iso-local}
PUT  /valet/validations/ticket-info/{ticket}?checkIn={iso-local}
```

**Headers requeridos**: `Operator-Id`, `Location-Id`.
**Path**: `{ticket}` (string).
**Query**: `checkIn` — string ISO en local time **sin offset**, tal cual lo devuelve el overnight summary (`2021-10-08T15:41:17.924107`).

**Response GET** — envuelto en `Response<T>`:
```json
{ "code": "OK", "data": { "vehicleInfo": {...}, "guestInfo": {...} } }
```

**Body PUT** — mismo shape que el `data` de la response:
```json
{
  "vehicleInfo": { "maker": "...", "model": "...", "color": "...", "plate": "..." },
  "guestInfo":   { "firstName": "...", "lastName": "...", "phoneCodeId": 1, "phoneNumber": 5511223344 }
}
```

`phoneCodeId` es el **countryId** (FK a `custom.cat_country.country_id`), no el dial number. Ver `backend/all/reference_phone_code_id_semantics.md`.

## Arquitectura backend

- **Controller**: `ValidationsController` (paquete `com.corepark.valet.validations.controller`). Convive con el ya existente `POST /validations/guest-page`.
- **Service**: `ValidationsServiceImpl.getTicketInfo` y `updateTicketInfo`. `@Transactional` en el update.
- **DAO**: `ValidationsDaoImpl` con SQL propio (SELECT + UPDATE) — sin reuso de `HotelDao`. Los WHERE filtran por `operator+location+ticket+check_in` (la PK real; ver `backend/all/reference_parking_service_row_identity.md`) y convierten el checkIn local a `TIMESTAMP WITH TIME ZONE` haciendo `JOIN parking_location ... AND ps.check_in = (:checkIn::timestamp WITHOUT TIME ZONE AT TIME ZONE pl.parking_location_tz)`.
- **DTOs**: `VehicleInfo`, `GuestInfo`, `TicketInfo` (bean único request+response) bajo `validations/beans/dto/`.
- **Firebase sync**: publica `MQTicket` a `firebase-realtime` con status `EDIT_TICKET_INFO`. Mismo status que ya usaba el hotel edit — `ms-firebase-service` no requiere cambios.

## Arquitectura frontend

- **Componente**: `shared/components/edit-ticket-info-dialog/` con schema propio (`*-schema.ts`), corepark-ui `DialogService`, `mat-form-field` + `matInput`, y `mat-autocomplete` para el phone code.
- **Trigger**: la columna `actions` del `overnight-report-summary` es ahora un solo `cp-menu` kebab (Detail, Request Car, Edit Info; y "Row Detail" en mobile). Reemplaza al layout mat-menu + buttons-container.
- **Service**: `ValidationService.getTicketInfo(ticket, checkIn)` y `updateTicketInfo(ticket, checkIn, payload)`. Manda `Operator-Id`/`Location-Id` en headers y `checkIn` como query param. `HttpPath.ValidationsTicketInfo = '/valet/validations/ticket-info'`.
- **Interceptor**: `/validations/ticket-info` está en `NOT_ALLOWED_PATH_LIST` de `operatorBodyInterceptor` para no duplicar operator/location en el body.
- **Catálogo de países**: `CatalogueService.getCountryCodes()` hace GET a `/valet/catalogs/country-codes` (endpoint dedicado del valet-service; cacheado in-memory desde el startup del JVM en `InitialCatalogLoading`). Se carga **lazy** al abrir la dialog vía `forkJoin` con el getTicketInfo, y se cachea en `CatalogueState.countries` (signal). Aperturas subsecuentes usan el cache — 0 network calls.
- **Autocomplete**: el form control `phoneCountry` guarda el objeto `Country | string | null`; `displayCountry` formatea como `+{phoneCode} {countryName}`; filtro por nombre, dial code o iso code. Prefill: buscamos por `countryId === guestInfo.phoneCodeId`.
- **Notificaciones**: usa `NotificationService` de corepark-ui. Wire-up global en `app.config.ts` con `provideNotifications({ position: 'top-right' })`.
- **Refresh post-save**: al cerrar la dialog con `true`, el summary re-lee `FullDateFilterState.signalFullDate()` y llama `OvernightReportService.getSummary(...)`, actualizando `OvernightReportState.tickets`. Los cambios se reflejan sin re-correr los filtros manualmente.

## Datos no obvios

- **`isEticket` no viaja al backend** en este endpoint. El backend edita los 8 campos igual; no hay lógica condicional para e-tickets (decisión explícita del usuario).
- **`requestPointId` tampoco viaja**. No se registra en `parking_service_info` como sí lo hacía el hotel edit — este endpoint es un UPDATE puro a `parking_service`, sin audit trail extra.
- **Row identity**: el PK real de `parking_service` es `(operator_company_id, parking_location_id, ticket, check_in)` — un mismo ticket string puede aparecer múltiples veces por operator+location con distintos check_ins. **Nunca** identifiques la fila solo por ticket.

## Ramas y merges (2026-08-06)

- Backend: `feature/validations-ticket-info` (desde main limpio) → merge en `feature/staging`.
- Frontend: `feature/edit-ticket` → merge en `develop`. Conflictos resueltos con la feature del Kiosk que evolucionó en develop (mobile-nav, navigation, option-icon). `package.json` combinó `--poll 2000 --no-hmr` global con el nuevo `start:local`.
- `ms-firebase-service`: sin cambios (el status `EDIT_TICKET_INFO` ya lo manejaba).

## Local dev setup asociado

Ver `frontend/validation/reference_local_dev_setup.md` para el detalle del script `start:local`, el `environment.local.ts` y el prefijo `/valet/` obligatorio en los HttpPath para que ruteen contra el gateway local.
