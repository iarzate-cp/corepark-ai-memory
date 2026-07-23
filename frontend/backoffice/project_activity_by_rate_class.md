---
name: Activity by Rate Class — v1 draft en feature/parking-volume-analytics (pendiente feedback)
description: Módulo unificado (serie + hourly-profile) implementado; v1 pusheado a staging esperando feedback QA/PO antes de iterar
type: project
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
Módulo bajo `/analytics/activity-by-rate-class` que consume dos endpoints del `ms-reports-service`:

- `POST /reports/analytics/tickets/parking-location/activity` — serie temporal (WEEK / DAY / HOUR); buckets con `checkIns`, `checkOuts`, `avgCarsOnHand`, cada uno con `total + byRateClass` (TRANSIENT / OVERNIGHT).
- `POST /reports/analytics/tickets/parking-location/hourly-profile` — 24h del día típico sobre un rango; mismo shape `total + byRateClass`.

Reemplazó dos páginas mock (`volume.ts` + `check-in-out.ts`) que compartían un `VolumeService` y un mock JSON de 1110 líneas — todo borrado.

**Estado v1 (pusheado a `feature/staging`):**
- Página `ActivityByRateClassPage` (glue) + shared component `ActivityByRateClassView` con date range picker en overlay CDK, toggles de sub-vista / granularidad / breakdown, dos charts stackeados por vista.
- Loader global + notificaciones success (con contexto: location, granularidad, rango, count) y error (mensajes user-friendly según código del enum `ActivityErrorCode`).
- Reset a semana en curso al cambiar granularidad + normalización enforzando reglas del backend (WEEK: semanas ISO completas; HOUR: `start === end`).
- `Operator-Id` + `Location-Id` van en headers; interceptor legacy tiene bypass en `EXCLUDED_URLS` para `/reports/analytics/`.
- Response envelope `{ code, message, ...payload }` (flat), no `{ data, code, message }`.
- Signals-first (computeds para location, rangeLabels, granularityLabel); `#locationId` como dep del request para que swap de parking lot refire fetch.
- i18n en/es con typed constants en `@i18n/analytics-i18n`.

**Pending para v1.1 después de feedback QA/PO:**
- Tests unitarios de `activity-by-rate-class-view-utils.ts` (bloqueador de convención CLAUDE.md).
- `docs/activity-by-rate-class.md` en español (convención hubspot-integration.md).
- Clamping proactivo de rango por granularidad usando `maxDays` del picker (evita `INVALID-DATE-RANGE` 400).
- Icono real (Tabler) en el trigger del picker en lugar del `▾` de texto.
- `aria-haspopup="dialog"` + `aria-expanded` en el trigger del dropdown.
- Chart tooltips con `tooltipData` contextual.
- QA en browser con data no-cero (todo lo que probamos fue con checkIns/checkOuts = 0).

**Iteración v2:**
- URL query params para rango/granularidad/vista (shareable links).
- Filtro por rate class específico (solo TRANSIENT / solo OVERNIGHT).
- Migrar de `dashboard-trend-chart` local a `LineChartComponent` de corepark-ui (soporta >2 series nativas).
- Presets del picker: "Últimos 30 días", etc.
