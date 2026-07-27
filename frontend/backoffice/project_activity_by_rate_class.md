---
name: Activity by Rate Class — dual-table redesign en feature/parking-volume-analytics
description: Reemplaza charts por dos tablas simultáneas; granularity dropdown solo en series; hourly no toma granularity
type: project
originSessionId: 5506e125-b5d9-4d50-a707-64609419d5d0
---
Módulo `/analytics/activity-by-rate-class` que consume dos endpoints de `ms-reports-service`:

- `POST /reports/analytics/tickets/parking-location/activity` — serie temporal (WEEK / DAY / HOUR); buckets con `{ periodStart, checkIns, checkOuts, avgCarsOnHand }`, cada métrica con `{ total, byRateClass: [{ rateClass, value }] }` (TRANSIENT / OVERNIGHT).
- `POST /reports/analytics/tickets/parking-location/hourly-profile` — 24h del día típico sobre un rango; shape hour: `{ hour, avgCheckIns, avgCheckOuts, avgCarsOnHand }` con el mismo `{ total, byRateClass }`.

**Rediseño actual (post-migración de charts a tables):**
- Dos cards **simultáneas** (no view-toggle): Volume by Rate Type (usa `/activity`) + Hourly Volume by Rate Type (usa `/hourly-profile`).
- Rows fijos: Transient / Overnight / Total (Total en `--color-main-500` + bold).
- Columns: buckets por granularity (series), 24 horas fijas (hourly).
- Métrica mostrada: `checkIns.byRateClass` en series, `avgCheckIns.byRateClass` en hourly. `checkOuts` y `avgCarsOnHand` NO están visibles (perdidos vs diseño viejo).
- Utils: `buildSeriesTable` / `buildHourlyTable` en `activity-by-rate-class-view-utils.ts`.

**Controles:**
- **Date range picker** (compartido) usa `cp-date-range-picker` del DS dentro de un CDK overlay via `cpDropdownTrigger`. Anchor a la derecha (`originX/overlayX: 'end'`).
- **Granularity dropdown** SOLO en series card. Usa DS `[cpMenuTrigger]` + `<cp-menu>` + `<cp-menu-item>`. Opciones: Daily / Weekly / Hourly (mapea al enum `ActivityGranularity`).
- **NO dropdown en hourly card** — quitado porque `/hourly-profile` no toma granularity, la UI antes engañaba al usuario.
- **NO view toggle, NO breakdown toggle** — obsoletos con el layout dual.

**Reactive pattern crítico (worth remembering):**
- `#range` signal contiene el rango CRUDO que el usuario picó. NO se normaliza al aplicar ni al cambiar granularity.
- `#activityRequest` computed normaliza el range localmente antes de mandarlo (`normalizeRangeForGranularity`). Depende de `#range` + `#seriesChoice` + `#locationId`.
- `#hourlyRequest` computed usa `#range` sin modificar. Depende SOLO de `#range` + `#locationId`.
- Consecuencia deseada:
  - Primera carga (locationId resuelve): ambos endpoints.
  - Cambio de fecha en picker: ambos endpoints.
  - Cambio de granularity: **solo `/activity`** (hourly no refetch porque su computed no depende de `#seriesChoice`).
- Implementación anterior mutaba `#range` en `onSeriesChoiceChange`, causando refetch spurio de hourly.

**Contrato HTTP (verificado, sin cambios desde v1):**
- Response envelope FLAT: `{ code, message, granularity, buckets }` (activity) / `{ code, message, dayCount, hours }` (hourly). NO `{ data, code, message }`. Ver `feedback_response_envelope_varies.md`.
- Headers manuales en el service: `Operator-Id` + `Location-Id` (no en body). `getActivity` / `getHourlyProfile` retornan `EMPTY` si falta operator o location.
- Interceptor legacy tiene `/reports/analytics/` en `EXCLUDED_URLS` para no inyectar `operatorCompanyId` al body. Ver `project_operator_id_header_pattern.md`.

**Reglas del backend enforzadas en `normalizeRangeForGranularity`:**
- WEEK: `startOf('week')` a `endOf('week').startOf('day')` — semanas ISO completas.
- HOUR: `start === end`, se colapsa a un solo día.
- DAY: `startOf('day')` en ambos endpoints.
- La normalización SOLO se aplica al construir el request de `/activity` (dentro del computed). No muta `#range`.

**Restricción por operador (commit `3d14700b`, sin cambios):**
- Guard: `core/guards/activity-by-rate-class/activity-by-rate-class.guard.ts` — usa `toObservable(operatorDataState.operatorCompanyId)` + `canAccessActivityByRateClass`.
- Predicate: `core/utils/activity-by-rate-class-access.ts` — `!environment.production` → true; en prod solo `ProductionOperators.CorePark` (1) y `SecureParking` (2).
- Nav filter: `main-layout-nav.routes.ts::getAppRoutes(operatorCompanyId)`.
- Wire-up: `app.routes.ts` aplica `canActivate: [activityByRateClassGuard]`.

**Animaciones (Angular Animations, no CSS):**
- Trigger `rangePanelFade` en el component metadata.
- `:enter` — opacity 0→1 + translateY(-4px → 0), 140ms ease-out.
- `:leave` — reverso, 120ms ease-in.
- Aplicado a un `<div @rangePanelFade>` DENTRO del `<ng-template>` del picker (el CDK crea un embedded view via TemplatePortal, así Angular detecta el `:enter`/`:leave`).

**Extensión al shared `CpDropdownTriggerDirective` (nueva):**
- Input `cpDropdownExitDelay = input(0)` (default 0 = backwards-compat).
- En `close()`: `detach()` síncrono (dispara `:leave`), luego `setTimeout(dispose, delay)` para dar tiempo a la animación de Angular antes de matar el pane host del CDK. Sin este delay, `dispose()` sync mata el DOM antes de que la animación empiece.
- Consumido por activity-by-rate-class con `[cpDropdownExitDelay]="120"` matcheando la duración del `:leave`.

**DS components importados desde `@corepark/corepark-ui`:**
- `ButtonDirective` (`[cpButton]`) — para el trigger del date picker (`variant="secondary" size="sm"`). Trigger del granularity dropdown NO usa cpButton (variant `text` es todo teal; el mock quería texto oscuro + caret teal — plain button).
- `DateRangePickerComponent` (`<cp-date-range-picker>`).
- `NotificationService`.
- `CpMenuComponent` / `CpMenuItemComponent` / `CpMenuTriggerDirective`.

**Layout / SCSS (dueño: usuario, no tocar sin pedirlo):**
- `:host` — flex column, gap 1rem, width 100%.
- `header` — flex row, justify-end (donde vive el date picker trigger).
- `.activity-by-rate-class-table-container` — flex column, gap 1.5rem, width 100%, padding 2rem, border-radius 1rem, background white, box-shadow.
- `.activity-view__card-header` — flex row space-between, width 100%. `<cp-menu>` DEBE estar fuera del header (sibling), no adentro, o rompe el space-between.
- Tabla: minimal styling, headers en `--color-grey-300`, cells en `--color-grey-500`, `tbody tr { border-top: 1px solid --color-grey-100 }`, fila `--total` en `--color-main-500` + bold.

**Mock deviations rechazadas (ver `feedback_mock_discipline.md`):**
- AM/PM toggle: nunca implementado, sin backing.
- Monthly granularity: enum del backend no tiene MONTH.
- Custom granularity: semantic ambigua (ya hay date picker).
- Dropdown en hourly card: removido (endpoint no toma granularity).

**Pending:**
- Métrica selector (checkOuts, avgCarsOnHand — hoy invisible).
- Tests unitarios de `activity-by-rate-class-view-utils.ts` (bloqueador de convención CLAUDE.md).
- `docs/activity-by-rate-class.md` en español.
- Icono real (Tabler) en trigger del picker en lugar de `▾`.
- `aria-haspopup="menu"` + `aria-expanded` en trigger del granularity dropdown.
- QA en browser con data no-cero.
- Migrar `NotificationService` como default sobre Angular Material snackbar (CLAUDE.md queda desactualizado aquí).
