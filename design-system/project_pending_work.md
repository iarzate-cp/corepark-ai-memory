---
name: Pending Work — corepark-ui integration (June 2026)
description: In-progress and deferred tasks across design-system, frontend-commerce and frontend-backoffice repos, with branch context
type: project
originSessionId: f9e7fb09-1e99-4725-9385-00220b3b086c
---

## Design-system repo — branch: `develop`

### Completado (2026-06-01)
- **`--cp-ui-*` prefix rename**: todas las CSS custom properties de la librería renombradas (430 ocurrencias en 31 archivos SCSS).
- **`DateRangePickerComponent` — overnight toggle**: inputs `showOvernight` y `overnightValue` añadidos.

### Completado (2026-06-09) — cp-table en módulos + fix doble línea en charts
- **`valet-overview` y `staff-roster`**: tablas migradas de `<table>` HTML crudo → `cp-table` + `CpCellDef`; estilos `.vo-table` / `.sr-table` eliminados
- **`cp-stacked-bar-chart`**: fix doble línea en eje X — grid line a tick=0 eliminada (`@if tick > 0`) + borde movido a `border-top` en `.sbc__x-row`
- **`cp-bar-chart`**: mismo fix — guard `@if (tick > 0)` en `<line class="grid-line">` del SVG para evitar overlap con `<line class="baseline">`
- **Versión 0.0.13 publicada** en GitHub Packages
- **`@corepark/corepark-ui`** bumpeado a `0.0.13` en frontend-commerce (con pnpm)

### Completado (2026-06-02) — Primitivos + módulos Valet Dashboard
- **`cp-avatar`**, **`cp-pill-group`**, **`cp-progress-bar`**, **`cp-pipeline`**, **`cp-bubble-chart`** — primitivos construidos
- **5 módulos completos** en `lib/modules/`:
  - `cp-module-location-occupancy` — grid de cards con donut + stats + leyenda
  - `cp-module-volume-trends` — pill-group toggle + line/bar chart + stat cards
  - `cp-module-valet-performance` — tabla con avatar + badge + progress bar + filtro por status
  - `cp-module-ev-charging-queue` — pipeline + tabla con búsqueda y paginación
  - `cp-module-duration-of-stay` — bubble chart + bar chart en grid 1fr 1fr
- **Responsive design en todos los módulos**:
  - `location-occupancy`: `≤1023px` → 2 cols (`min()` CSS); `≤639px` → 1 col + body apila verticalmente
  - `volume-trends` / `duration-of-stay`: `≤1023px` → 2 cols; `≤639px` → 1 col
  - `ev-charging-queue`: `overflow: hidden; min-width: 0` en el wrapper del pipeline para contener el scroll horizontal sin filtrar al page
- **Versión 0.0.12 publicada** en GitHub Packages (`npm publish` desde `dist/corepark-ui/`) → actualizada a 0.0.13 en sesión 2026-06-09

### Pendiente — design-system
- `sidebar-component.ts` Tailwind → BEM migration (branch `chore/sidebar-scss-migration`, deferred)
- Spec files: 17+ archivos `*.spec.ts` sin commitear

---

## Frontend-commerce repo — branch activo: `implementation/corepark-ui` (mergeado a HEAD)

### Completado (2026-06-02)
- **Valet Dashboard page** (`/valet-dashboard`) construida con los 5 módulos de la librería
- **Fix horizontal overflow** en layout del commerce:
  - `.valet-content`: `min-width: 0; overflow-x: hidden` añadidos
  - `.dashboard-section` + `.dashboard-row`: `min-width: 0` añadidos
- **Nav — sección VALET**: item `Dashboard → /valet-dashboard` con `chart-pie-icon` añadido a `main-layout.component.ts`
- **`ui-components` showcase** expandido con 6 secciones nuevas: Avatar, Alert, Progress Bar, Pill Group, Pipeline, Table
- **`@corepark/corepark-ui`** bumpeado a `0.0.12` en `package.json`
- **Conflicto de merge resuelto** en `main-layout.config.ts` y `main-layout.component.html`: HEAD tenía `password-user` (ACCESS CONTROL section); nuestra rama tenía `chart-pie` (VALET section) — se conservaron ambos

### Completado (2026-06-09)
- **`@corepark/corepark-ui`** bumpeado a `0.0.13` vía pnpm; `pnpm-lock.yaml` actualizado

### Pendiente — commerce
- *(sin pendientes activos)*

---

## Frontend-backoffice repo — branch: `develop`

### Completado (2026-06-01) — Sesión 1: ticket-log CDK overlay
- **`TicketLogDateFilter`** — CDK overlay con `cp-date-range-picker` inline
- **`ticket-log.component.ts`** simplificado: eliminado `MatDialog`, `FullDateFilterLayoutState`
- **`angular.json`** styles: `@corepark/corepark-ui/styles.css` añadido como primer entry

### Completado (2026-06-01) — Sesión 2: DateFilterButton + migración global de reportes
- **`DateFilterButton`** — nuevo componente genérico en `shared/components/date-filter-button/`
- **7 páginas de reportes migradas** de `MatDialog.open(FullDateFilterComponent)` → `DateFilterButton`

### Completado (2026-06-01) — Sesión 3: mobile overlay + navegación mes/año + bug fixes picker
- **`cp-date-range-picker`** — navegación mes/año/década (3 vistas: days → months → years)
- **`DateFilterButton` + `TicketLogDateFilter` mobile**: migrados de `CdkConnectedOverlay` → `Overlay` service imperativo
- **`cp-date-range-picker` responsive** (design-system): layout mobile en columna, presets grid 3 cols

### Pendiente — backoffice
- Reemplazar `mat-select` en `FilterDialogComponent` con `cp-select`
- Migrar `DashboardComponent` de `MatDialog` → `DialogService`
- Migración completa de todos los dialogs restantes de `MatDialog` → `DialogService`
- Múltiples archivos con señales que aún usan `.getValue()` / `.asObservable()`
- Migrar `full-date-filter.component.ts` internamente (aún usa `nxt-pick-datetime`)

---

## Decisiones arquitectónicas confirmadas

- **`wrapper-dialog` y `dialog-wrapper` NO migran a `cp-dialog-content` todavía** — son usados por ~60+ dialogs.
- **`data-theme="dark"` NO va en `index.html` del backoffice** — área de contenido clara; tokens DS renderizan modo claro por defecto.
- **`styles.css` del DS debe ir PRIMERO** en el array de styles de `angular.json`.
