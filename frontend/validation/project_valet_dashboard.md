---
name: Valet Dashboard — estado actual e implementación
description: Qué está implementado, qué está pendiente, decisiones técnicas y detalles de integración del módulo Valet Dashboard
type: project
originSessionId: b01d2c86-5a0a-4a17-8365-d2a742c0d838
---
## Estado actual (2026-06-17) — V5

Rama: `feature/valet-dashboard`. Tres tabs: **Overview → Service Times → Staff**.

---

## Tab: Overview

### Endpoints (forkJoin en `valet-analytics-state.ts`)
- `POST /reports/analytics/tickets/parking-location/daily` → trends, KPIs, tabla day-by-day (**V5 rename** de `/activity/daily`)
- `POST /reports/analytics/tickets/parking-location/hourly` → trend por hora (**V5 rename** de `/activity/hourly`)
- `POST /reports/analytics/tickets/durations` → distribution de duraciones + summary
- `POST /reports/analytics/employees/activity` → staff roster
- `POST /reports/analytics/tickets/service-times/parking-areas` → Service Times (garajes)
- `POST /reports/analytics/tickets/service-times/daily` → Service Times (por día)
- `POST /reports/analytics/tickets/parking-areas/daily` → Daily Activity Log (**V5 nuevo**)

Todos corren en un solo `forkJoin`. Cada observable tiene su propio `catchError` → `#logError` → `console.error`:
```
[Valet Analytics] POST /reports/analytics/<endpoint>
{ headers: { 'Operator-Id', 'Location-Id' }, body: { startDate, endDate }, status, response }
```

### KPIs derivados client-side
- `activeStaff` = `employees.length` (no hay endpoint dedicado)
- `lowDay` excluye días con `checkIns === 0`
- `peakOccupancy` sin `checkIns > 0` muestra valor pero sin hora (`time: ''`)

### Staff roster (`cp-module-staff-roster`)
- Roles inferidos del mix de eventos: `frontline | dispatcher | checkout | generalist | light`
- Umbral de actividad baja: `eventCount < 20` → `role: 'light'`
- StatusIds mapeados: checkIn=7, park=5, damage=21, attend=2, complete=3, relocate=6
- `highlightColumn` aplica color warning (dispatcher) o purple (checkout)
- `filterable=true`, `showPagination=true`, `pageSize=15`
- Sort en: Name, Total, Check-In, Park, Damage, Attend, Complete, Relocate

### Tabla day-by-day (overview) — lógica dinámica
- `tableFilterable` y `tableShowPagination` se activan solo cuando `days.length > 14`
- `tablePageSize=14` (2 semanas por página cuando aplica)
- Sort en: Vehicles, Peak Occupancy

### Vehicle Mix y EV Chargers
Siguen con datos mock (`OVERVIEW_VEHICLE_MIX`, `OVERVIEW_EV_CHARGERS`). No hay endpoints definidos aún.

---

## Tab: Service Times (V4, 2026-06-12)

### Endpoints
- `POST /reports/analytics/tickets/service-times/parking-areas` → `{ parkingAreas[] }`
- `POST /reports/analytics/tickets/service-times/daily` → `{ days[] }`

### Señales en `valet-analytics-state.ts`
- `#stParking` → `serviceTimeStations` (computed via `mapServiceTimesParkingAreas`)
- `#stDaily` → `serviceTimeWaitDays` (computed via `mapServiceTimesWaitDays`)
- `#stDaily` → `serviceTimeDailyRows` (computed via `mapServiceTimesDaily`)

### Mapeo (`valet-analytics-utils.ts`)

**`mapServiceTimesParkingAreas(areas[]) → ServiceStation[]`**
- `name`: `parkingArea ?? 'Sin asignar'`
- `retrievalMin`, `dropOffMin`, `responseMin`: `Math.round(seconds / 60)` (sin cap — solo para los bars)
- `tier`: `'preliminary'` si `deliveredRequestCount < 10`; sino `'high'` si `retrievalMin > 60`, `'med'` si `> 30`, `'low'` resto
- `ticketCount` y `deliveredCount` para las cards de Garage Performance
- `parkingAreaId === null` → `note: 'Data legacy · sin parking area asignado'` + clase `st-garage-card--legacy`

**`mapServiceTimesWaitDays(days[]) → ServiceWaitDay[]`**
- Para el stacked bar chart (eje Y en minutos)
- Usa `p50ToMin(seconds, cap=true)` → **cap de 1440 min (24h)**
- **Por qué el cap:** en datos de test/dev aparecen P50 de 85-121 horas (tickets con arrival y park/retrieval separados por días). Sin cap, la escala Y del chart colapsa todos los valores reales al fondo. Un P50 > 24h es prácticamente siempre dato corrupto para una operación valet.

**`mapServiceTimesDaily(days[]) → ServiceDailyRow[]`**
- Para la tabla diaria (sin columna Total — eliminada en V4)
- `formatP50(seconds)`: `null` → `'—'`; `< 60s` → `Xs`; `< 3600s` → `Xm Ys`; `>= 3600s` → `Xh Ym` (horas para valores extremos)
- `highlight: true` si `deliveredRequestCount > 0 && < 10` (marca como preliminary)

### Semántica del modelo (V4 handoff)
- **Drop-off** = tiempo desde CHECK-IN/RE_CHECK_IN hasta el primer PARK posterior. Cada arrival cuenta como medición separada (multi-cycle hoteles). RELOCATE NO cuenta.
- **Response** = REQUEST → primer evento de reconocimiento (ATTEND=2, ACCEPT=4, EN_ROUTE=23, HOLD=24)
- **Retrieval** = reconocimiento → MIN(READY_FOR_PICKUP=20, TEMPORAL_CHECKOUT=13, COMPLETE=3)
- **deliveredRequestCount**: ciclos con TEMPORAL_CHECKOUT o COMPLETE. Sample size operativa.
- `deliveredRequestCount < 10` → Preliminary (datos insuficientes para P50 confiable)

### Cambios en corepark-ui para Service Times
- `ServiceStation`: añadidos `ticketCount?` y `deliveredCount?`
- `ServiceDailyRow`: `total` ahora opcional; columna eliminada de `dailyCols` y del template
- Template: nueva sección "Garage Performance" (cards responsive) antes de `st-grid`
- Label renombrado: "Retrieval Time by Station" → "Retrieval Time by Garage"
- SCSS: `.st-garage-card`, `.st-garage-cards`, `.st-garage-card--legacy`

---

## Date picker (`cp-date-range-picker`)

- Vive encima del `cp-tab-group`, aplica a todos los tabs
- Default: últimos 7 días (`now - 6 días` → `now`)
- Título "Valet Dashboard" como `<h1>` a la izquierda
- **`[maxDays]="90"`** — límite de 90 días (pendiente de ajuste con datos reales)
  - Días fuera del límite aparecen deshabilitados en el calendario
  - Hover preview se trunca al límite visualmente
  - Presets cuyo rango calculado supera `maxDays` se ocultan automáticamente (`filteredPresets` computed)
  - El hint "MAX 90 DAYS" aparece debajo del header de fechas
  - **Por qué 90 días:** rangos muy largos producen demasiadas barras en el stacked bar chart (chart ilegible con 100+ días) y el backend tarda > 3s en sites grandes (nota del handoff V4)

---

## StackedBarChartComponent — mejora de escala Y

`#niceStep(max)` calcula el step dinámicamente apuntando a ~7 ticks:
- `raw = max / 7`, luego redondea al número "nice" más cercano (1, 2, 5, 10 × magnitud)
- Reemplaza el switch hardcodeado `<20→5, <60→10, <120→20, else→50` que generaba 29+ ticks con max=1440

Ejemplos: max=1440→step=200 (8 ticks), max=120→step=20 (7 ticks), max=20→step=5 (5 ticks).

---

## Control de acceso — `valetDashboardGuard`

- **Dev** (`environment.production === false`): siempre permite acceso
- **Prod**: whitelist de pares `(operatorCompanyId, parkingLocationId)`:
  - `ParkingCompanyOfAmerica (274)` + `TeslaPaloAlto (4)`
  - `AllAboutParking (343)` + `Usds (1)` — añadido 2026-07-17
- Enums: `ProdOperatorsEnum` en `core/enums/prod-operators-enum.ts`, `ProdLocationsEnum` en `core/enums/prod-locations-enum.ts`
- La lógica vive en `OperatorDataState.showValetDashboard` como computed — el guard solo lee ese signal

---

## Layout (`valet-layout`)

- `cp-isotype` requiere `[filled]="true"` + variables CSS `--color-primary` y `--color-primary-lighter` mapeadas a los tokens `--cp-ui-color-*` dentro de `.valet-header` — sin esto el logo se ve negro
- Header sin nombre de módulo en el shell
- **Sombra en scroll**: header recibe clase `valet-header--scrolled` cuando `.valet-content` tiene `scrollTop > 0`. Requiere `z-index: 1` en el header — sin él el contenido (position: relative) tapa la sombra

---

## Definiciones de tipos (`core/definitions/analytics.d.ts`)

- `DurationBucket`: `id`, `lowerSeconds`, `upperSeconds` (null en último bucket), `ticketCount`, `ticketShare`, `staySeconds`, `stayShare`
- `EmployeeActivity`: `employeeId`, `firstName`, `lastName`, `eventCount`, `eventCounts[]`
- `ServiceTimeParkingArea`: `parkingAreaId | null`, `parkingArea | null`, `ticketCount`, `dropOffP50Seconds | null`, `responseP50Seconds | null`, `retrievalP50Seconds | null`, `deliveredRequestCount`
- `ServiceTimeDailyEntry`: `businessDate`, `dropOffP50Seconds | null`, `responseP50Seconds | null`, `retrievalP50Seconds | null`, `deliveredRequestCount`
- Bucket "workday" identificado por `lowerSeconds === 21600 && upperSeconds === 86400` (no por `id`, no es estable)

---

## Widget: Daily Activity Log (V5)

- Endpoint: `POST /reports/analytics/tickets/parking-areas/daily`
- Señal: `#parkingAreas` en state; computed `dailyActivityLog` via `mapDailyActivityLog()`
- Mapeo en `valet-analytics-utils.ts`: une `parkingAreasDays` + `flowDailyDays` (medianStay)
- Columnas dinámicas = unión de todas las parking areas del rango; "Unassigned" (`parkingAreaId=null`) siempre al final (key `'__unassigned__'`)
- Filas agrupadas por ISO calendar week en cliente (Luxon `weekYear + weekNumber`)
- Week label formato: `WEEK {n} (MAY 21–22) — 303 VEHICLES`
- Colores de columnas: `--cp-ui-color-primary`, `--cp-ui-color-warning`, `--cp-ui-color-info`, `--cp-ui-color-text-900`, … (cycling); unassigned = `--cp-ui-color-text-400`
- RECORD badge en el día con mayor total (dorado, `color-mix` de warning)
- Semántica: mide **presencia al cierre del día** (no check-ins del día). `ticketsOnHand ≠ checkIns`
- Componente: `cp-module-daily-activity-log` en corepark-ui `modules/daily-activity-log`

## Pendiente

- **Vehicle Mix y EV Chargers**: datos mock, sin endpoints
- **Ajuste del cap de 1440 min y maxDays=90**: pendiente de validar con datos reales de prod
- **Descripción del staff roster**: texto hardcodeado con nombres de ejemplo, necesita ser dinámico o eliminarse
