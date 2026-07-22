---
name: corepark-ui — Capa de Módulos
description: Nueva capa lib/modules/ con secciones de dashboard pre-ensambladas a partir de primitivos de la librería; exportables a todos los proyectos CorePark
type: project
originSessionId: f9e7fb09-1e99-4725-9385-00220b3b086c
---
## Qué es y por qué existe

La librería tiene dos capas:
- `lib/components/` — primitivos reutilizables (botones, charts, form controls, etc.)
- `lib/modules/` *(nueva)* — secciones de dashboard completas construidas **exclusivamente con primitivos de la librería**

**How to apply:** Cuando se construya un módulo, usar siempre componentes del propio DS. Nunca importar componentes del backoffice ni CSS externo.

---

## Referencia de diseño

Archivo HTML original: `/Users/israel/Downloads/tesla-valet-dashboard.html`
Dashboard: **Valet Operations Analytics** — 5 secciones principales.

---

## Estado actual (2026-06-09) — TODOS LOS MÓDULOS COMPLETOS + REFINADOS

### Primitivos disponibles para módulos
| Componente | Selector | Notas |
|---|---|---|
| AvatarComponent | `cp-avatar` | Iniciales auto, color configurable, sizes: sm/md/lg (no xs) |
| PillGroupComponent | `cp-pill-group` | Segmented control single-select |
| ProgressBarComponent | `cp-progress-bar` | Auto threshold coloring |
| PipelineComponent | `cp-pipeline` | Flujo horizontal de etapas con contadores y conectores |
| DonutChartComponent | `cp-donut-chart` | Segmentos con leyenda |
| LineChartComponent | `cp-line-chart` | Multi-series, área opcional |
| BarChartComponent | `cp-bar-chart` | Multi-series, grouped |
| BubbleChartComponent | `cp-bubble-chart` | Scatter con tamaño variable de burbuja |
| BadgeComponent | `cp-badge` | Status badges |
| CpTableComponent | `cp-table` | Tabla con sort/filter/paginación; overflow-x: auto interno |
| StatCardComponent | `cp-stat-card` | KPI card con variantes |

---

## Módulos completados — `lib/modules/`

### 1. `location-occupancy` — `cp-module-location-occupancy`
- Grid de cards (CSS custom prop `--loc-count` via host binding)
- Cada card: accent bar de color, donut chart (size=110, thickness=11), stats en lista, leyenda
- Responsive: `≤1023px` → `min(2, var(--loc-count))` cols; `≤639px` → 1 col, body apila verticalmente

### 1b. `valet-overview` (dentro de location-occupancy o módulo propio)
- Tabla de daily breakdown migrada a `cp-table` + `CpCellDef` (2026-06-09)
- Celdas condicionales: `vo-cell--bold` (fecha, peak time, median stay) y `vo-cell--primary` (vehicles, peak occupancy en filas highlight)
- `voIsHighlight(row)` detecta la fila resaltada; `voCell(row, key)` extrae el valor como string

### 2. `volume-trends` — `cp-module-volume-trends`
- Pill-group toggle line/bar chart + stat cards en grid (`--stat-count` via host binding)
- Badge de tendencia con variante `success/danger` según sign del trend
- Responsive stats: `≤1023px` → 2 cols; `≤639px` → 1 col

### 2b. `staff-roster` — `cp-module-staff-roster`
- Tabla de staff migrada a `cp-table` + `CpCellDef` (2026-06-09)
- Celdas condicionales: `sr-cell--num` (rank), `sr-cell--name`, `sr-cell--bold` (total), `sr-cell--highlighted` (attend/complete)
- `srHighlightColor(row, column)` devuelve color según role (dispatcher → warning, checkout → purple)
- Badge de rol: `sr-badge sr-badge--{role}` (frontline/dispatcher/checkout/generalist/light)

### 3. `valet-performance` — `cp-module-valet-performance`
- Filtro por status (pill-group) + tabla con avatar, tiempo avg, idle %, rating (progress-bar), status (badge)
- `ratingVariant()`: ≥80 → success, 60–79 → warning, <60 → danger
- `cp-table` ya tiene `overflow-x: auto` interno — no requiere wrapper extra

### 4. `ev-charging-queue` — `cp-module-ev-charging-queue`
- Pipeline + tabla con búsqueda y paginación
- Wrapper `__pipeline` tiene `overflow: hidden; min-width: 0` para contener el scroll horizontal del pipeline

### 5. `duration-of-stay` — `cp-module-duration-of-stay`
- Stats en grid (`--stat-count`) + dos charts (bubble + bar) en grid `1fr 1fr`
- Charts colapsan a `1fr` en `≤768px`

---

## Convenciones para módulos

- Ruta: `lib/modules/<nombre>/`
- Cada módulo: carpeta + `.component.ts` + `.component.html` + `.component.scss` + `index.ts`
- Export desde `lib/modules/index.ts` → re-export en `public-api.ts`
- Standalone, `ViewEncapsulation.None`, BEM con prefijo `cp-module-*`
- Inputs: datos crudos (arrays de objetos); el módulo no hace HTTP
- CSS custom properties para grids dinámicos: setear vía `host: { '[style.--N]': 'computed()' }`
- Responsive: breakpoints `@media (max-width: 1023px)` (tablet) y `@media (max-width: 639px)` (mobile)
