---
name: Activity by Rate Class — feature/parking-volume-analytics (5 commits ahead de main)
description: Módulo unificado que reemplaza volume + check-in-out; v1 en review con QA/PO, restricción por operador ya aplicada
type: project
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
Módulo `/analytics/activity-by-rate-class` que consume dos endpoints de `ms-reports-service`:

- `POST /reports/analytics/tickets/parking-location/activity` — serie temporal (WEEK / DAY / HOUR); buckets con `{ periodStart, checkIns, checkOuts, avgCarsOnHand }`, cada métrica con `{ total, byRateClass: [{ rateClass, value }] }` (TRANSIENT / OVERNIGHT).
- `POST /reports/analytics/tickets/parking-location/hourly-profile` — 24h del día típico sobre un rango; shape hour: `{ hour, avgCheckIns, avgCheckOuts, avgCarsOnHand }` con el mismo `{ total, byRateClass }`.

Reemplazó dos páginas mock (`volume.ts` + `check-in-out.ts`) que compartían un `VolumeService` y un mock JSON de 1110 líneas — todo borrado.

**Estado actual (branch `feature/parking-volume-analytics`, 5 commits ahead de main, remoto existe):**
- Página glue: `src/app/pages/main/analytics/activity-by-rate-class.ts` (`ActivityByRateClassPage`) — solo `<wrapper>` + `<activity-by-rate-class-view />` + reset del loader.
- Shared component: `src/app/shared/components/activity-by-rate-class-view/` (component .ts + .html + .scss + view-utils.ts + index).
- Service: `src/app/core/services/reports/activity-by-rate-class/activity-by-rate-class-service.ts` (barrel + folder — la convención "flat" de CLAUDE.md no se respetó aquí, pero es lo que hay).
- Definitions: `src/app/core/definitions/analytics/activity-by-rate-class.d.ts`.
- Enums: `activity-error-code.ts`, `activity-granularity.ts` (WEEK/DAY/HOUR), `rate-type.ts` (TRANSIENT/OVERNIGHT).
- Endpoints en `core/http/endpoints/analytics.ts`, agregado a `endpoints.ts`.
- i18n typed constants: `@i18n/analytics-i18n` — keys presentes en `assets/i18n/en.json` y `es.json`.

**Restricción por operador (commit `3d14700b`, post-snapshot anterior):**
- Guard: `core/guards/activity-by-rate-class/activity-by-rate-class.guard.ts` — usa `toObservable(operatorDataState.operatorCompanyId)` + `canAccessActivityByRateClass`; redirige a `/analytics/trend` si no aplica.
- Util predicate: `core/utils/activity-by-rate-class-access.ts` — `!environment.production` → true; en prod solo `ProductionOperators.CorePark` (1) y `SecureParking` (2).
- Nav filter: `main-layout-nav.routes.ts::getAppRoutes(operatorCompanyId)` — remueve la entrada del urlTree de Analytics si el operador no aplica (defense-in-depth junto al guard).
- Wire-up: `app.routes.ts` aplica `canActivate: [activityByRateClassGuard]` en el path `analytics/activity-by-rate-class`.

**Contrato HTTP verificado:**
- Response envelope FLAT: `{ code, message, granularity, buckets }` (activity) / `{ code, message, dayCount, hours }` (hourly). NO `{ data, code, message }`. Ver `feedback_response_envelope_varies.md`.
- Headers manuales en el service: `Operator-Id` + `Location-Id` (no en body). `getActivity` / `getHourlyProfile` retornan `EMPTY` si falta operator o location.
- Interceptor legacy tiene `/reports/analytics/` en `EXCLUDED_URLS` para no inyectar `operatorCompanyId` al body. Ver `project_operator_id_header_pattern.md`.

**Reglas del backend enforzadas en el frontend (`normalizeRangeForGranularity`):**
- WEEK: `startOf('week')` a `endOf('week').startOf('day')` — semanas ISO completas.
- HOUR: `start === end`, se colapsa a un solo día.
- DAY: `startOf('day')` en ambos endpoints.
- `onGranularityChange` SIEMPRE resetea al rango de la semana actual normalizado — decisión intencional para evitar rangos inválidos al cambiar granularidad.

**Patrones internos del view component (worth remembering):**
- Request derivation: `#activityRequest` y `#hourlyRequest` son `computed(...)` que retornan `null` cuando las precondiciones no se cumplen (view mode incorrecto, sin location, sin range). El pipeline es `toObservable(computed).pipe(filter(non-null), switchMap(service.get))` + `toSignal`. Así, cualquier cambio en dependencias (location, range, granularity, view) re-dispara la request correcta sin lógica manual.
- Notificaciones: `effect() { untracked(() => this.#notifySeriesUpdated(...)) }` para que el toast no cree dependencias reactivas.
- Toasts vienen de `NotificationService` de `@corepark/corepark-ui`, no de Angular Material snackbar (CLAUDE.md queda desactualizado en este punto).
- Chart component: usa `dashboard-trend-chart` local (max 2 series) — de ahí el ítem "migrar a `LineChartComponent` de corepark-ui" en el backlog v2.

**Breakdown modes:**
- `totals`: 2 charts (check-in vs check-out combinados, avg cars solo).
- `byRateClass`: 3 charts (checkIns, checkOuts, avgCarsOnHand) cada uno con transient vs overnight.

**Pending para v1.1 después de feedback QA/PO:**
- Tests unitarios de `activity-by-rate-class-view-utils.ts` (bloqueador de convención CLAUDE.md).
- `docs/activity-by-rate-class.md` en español (convención hubspot-integration.md).
- Clamping proactivo de rango por granularidad usando `maxDays` del picker (evita `INVALID-DATE-RANGE` 400).
- Icono real (Tabler) en el trigger del picker en lugar del `▾` de texto.
- `aria-haspopup="dialog"` + `aria-expanded` en el trigger del dropdown.
- Chart tooltips con `tooltipData` contextual.
- QA en browser con data no-cero (todo lo probado tuvo checkIns/checkOuts = 0).

**Iteración v2 (backlog):**
- URL query params para rango/granularidad/vista (shareable links).
- Filtro por rate class específico (solo TRANSIENT / solo OVERNIGHT).
- Migrar de `dashboard-trend-chart` local a `LineChartComponent` de corepark-ui (soporta >2 series nativas).
- Presets del picker: "Últimos 30 días", etc.
