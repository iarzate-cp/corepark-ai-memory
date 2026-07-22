---
name: Design system — workflow de build y sync
description: Cómo modificar corepark-ui, buildearlo y sincronizarlo con frontend-validation
type: reference
originSessionId: 8f161420-0860-4a92-a4cd-a5269c24d5a4
---
## Repositorio

`/Users/israel/Dev/design-system` — paquete `@corepark/corepark-ui` v0.0.14

El paquete está instalado vía pnpm en frontend-validation pero **no es un symlink al source local** — hay que buildearlo y sincronizarlo manualmente.

## Workflow completo

```bash
# 1. Modificar source
# /Users/israel/Dev/design-system/projects/corepark-ui/src/lib/...

# 2. Build
cd /Users/israel/Dev/design-system && npm run build

# 3. Sync a frontend-validation
npm run sync:validation
```

## Scripts disponibles

- `npm run build` — compila Angular + tokens SCSS + styles CSS + fix paths
- `npm run sync:validation` — rsync dist → frontend-validation node_modules
- `npm run sync:backoffice` — rsync dist → frontend-backoffice node_modules
- `npm run sync:commerce` — rsync dist → frontend-commerce node_modules
- `npm run publish:lib` — publica a npm registry

## Ubicación de módulos relevantes

- Tabla: `projects/corepark-ui/src/lib/components/table/`
- Staff roster: `projects/corepark-ui/src/lib/modules/staff-roster/`
- Valet overview: `projects/corepark-ui/src/lib/modules/valet-overview/`
- Bar chart: `projects/corepark-ui/src/lib/components/charts/bar-chart/`

## Cambios acumulados en corepark-ui (2026-06-12)

### `cp-module-staff-roster`
Nuevos inputs (todos con defaults que preservan comportamiento anterior):
- `filterable: boolean` (default `false`)
- `showPagination: boolean` (default `false`)
- `pageSize: number` (default `25`)
- Columnas numéricas y name con `sortable: true`

### `cp-module-valet-overview`
Nuevos inputs:
- `tableFilterable: boolean` (default `false`)
- `tableShowPagination: boolean` (default `false`)
- `tablePageSize: number` (default `25`)
- Columnas Vehicles y Peak Occupancy con `sortable: true`

Scroll horizontal en portrait mobile para el chart:
- `.vo-trend__chart-inner` wrapper añadido en el template
- Outer (`.vo-trend__chart`): `overflow-x: auto; overflow-y: hidden; height: 18.75rem` en portrait
- Inner (`.vo-trend__chart-inner`): `min-width: 75rem; height: 18.75rem` en portrait
- **Nota crítica**: height explícita en el inner es necesaria — `height: 100%` no resuelve cuando el padre tiene `overflow-y: hidden`

### `cp-module-service-times`
Nuevos campos en interfaces (todos opcionales, backward compatible):
- `ServiceStation.ticketCount?` y `ServiceStation.deliveredCount?` — para las cards de Garage Performance
- `ServiceDailyRow.total?` — campo hecho opcional; columna eliminada del `dailyCols` y del template

Cambios en template:
- Nueva sección "Garage Performance" (grid de cards) encima del `st-grid`
- Label "Retrieval Time by Station" → "Retrieval Time by Garage"
- Eliminado el `ng-template cpCell="total"` y la columna del `dailyCols`

Nuevos estilos SCSS: `.st-garage-cards`, `.st-garage-card`, `.st-garage-card--legacy`

### `cp-date-range-picker`
Nuevo input `maxDays: number | null` (default `null`):
- `filteredPresets` computed: oculta presets cuyo rango supera `maxDays` (calculado dinámicamente — e.g., "This year" desaparece si hoy > día 89 del año)
- `calendarWeeks`: días posteriores a `start + maxDays - 1` aparecen con `isDisabled: true`
- `onDayPointerEnter`: hover preview clampeado al límite
- Hint visual "MAX N DAYS" debajo del header de fechas (estilo `.drp__limit-hint`)
- Nota: `onPreset` ya no necesita clamp propio — los presets fuera del límite no se muestran

### `cp-stacked-bar-chart`
Reemplazado el step hardcodeado (`<20→5, <60→10, <120→20, else→50`) por `#niceStep(max)`:
- Calcula `raw = max / 7`, luego redondea al "nice number" más cercano (1, 2, 5, 10 × magnitud de 10)
- Siempre produce ~7 ticks independientemente del rango
- Motivación: con cap de 1440 min en Service Times, el step=50 anterior generaba 29 ticks apilados ilegibles
