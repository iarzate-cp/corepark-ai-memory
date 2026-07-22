---
name: Date Picker Integration Plan — backoffice
description: Plan para migrar nxt-pick-datetime a cp-date-range-picker en el backoffice; estado actual, diferencias de API, sitios de uso, bugs a corregir primero, conversión de tipos
type: project
originSessionId: f1d14aab-a36f-4dc1-b961-fde3c10a1821
---

## Objetivo

Integrar `cp-date-range-picker` del design system en `/Users/israel/Dev/frontend-backoffice`, reemplazando `nxt-pick-datetime`. El alcance inicial es solo el date picker — no migrar otros componentes.

**Why:** Consolidar en la librería propia, eliminar dependencia de terceros, tener control total sobre UI y comportamiento.

**How to apply:** Cuando se pida implementar o avanzar con el date picker en el backoffice, usar este documento como punto de partida — ya tiene el estado real del código.

---

## Estado actual del backoffice

- `@corepark/corepark-ui` **v0.0.10 ya está instalada** en node_modules
- `nxt-pick-datetime` v21.0.0 es la librería actual; registrada globalmente en `app.config.ts` via `importProvidersFrom(DateTimeModule, NativeDateTimeModule)`
- `angular.json` styles array incluye `"node_modules/@corepark/corepark-ui/styles.css"` como primer entry (necesario para los tokens `--cp-ui-*`)

---

## MIGRADO: ticket-log → `TicketLogDateFilter` (2026-06-01) ✅

**Sitio**: `pages/main/reports/ticket-log/ui/ticket-log-date-filter/`

**Patrón usado**: CDK `CdkConnectedOverlay` (NO `MatDialog`) — `cp-date-range-picker` inline dentro de un panel dropdown animado anclado al botón filtro.

---

## MIGRADO: 7 páginas de reportes → `DateFilterButton` (2026-06-01) ✅

**Patrón**: componente genérico `shared/components/date-filter-button/` con CDK overlay + `cp-date-range-picker`.

Inputs disponibles:
- `showOptions = input(false)` — muestra los 3 checkboxes (Inventory, Departed, Temp checkout)
- `showOvernight = input(false)` — muestra el toggle overnight y setea `isTicketTransactionsOvernight`

Páginas migradas:
- `validations-actions` — `<date-filter-button />`
- `cash-reconcile` — `<date-filter-button />`
- `ticket-transactions` — `<date-filter-button [showOvernight]="true" />`
- `daily` — `<date-filter-button />`
- `tip-report` — `<date-filter-button />`
- `time-tracking` — `<date-filter-button />`
- `overnight` — `<date-filter-button [showOptions]="true" />`

---

## PENDIENTE: `full-date-filter.component.ts` (usa `nxt-pick-datetime` internamente)

Este componente ya NO se usa como `MatDialog` en ninguna de las páginas migradas arriba, pero el componente en sí (`shared/components/full-date-filter/`) sigue existiendo y usa `nxt-pick-datetime` internamente. No se ha migrado su interior.

---

## Diferencia de patrón de uso (crítica)

`nxt-pick-datetime` = **trigger + overlay** sobre un `<input>`. El picker se abre al hacer click en el input.

`cp-date-range-picker` = **panel inline siempre visible**. No tiene trigger propio — se embebe directamente en un dialog, dropdown o panel.

Esto significa que la migración no es un swap de directivas: los componentes que hoy usan `nxt-pick-datetime` dentro de un `MatDialog` deben reemplazar el `mat-form-field` + trigger por `<cp-date-range-picker>` directamente como contenido del dialog.

---

## Diferencias de API

| Concepto | `nxt-pick-datetime` | `cp-date-range-picker` |
|---|---|---|
| Tipo de fecha | `Date[]` (array nativo JS) | `DateRange { start: DateTime, end: DateTime }` (Luxon) |
| Binding | `FormControl<Date[]>` | input `value` / output `apply` |
| Opciones extra | no tiene | input `options: PickerOption[]` (checkboxes en footer) |
| Overnight | control separado | input `showOvernight` / `overnightValue`; emitido en `apply` |
| Limpiar | no tiene clear propio | output `clear` |
| Tiempo | `[hour12Timer]="true"` | time picker integrado, siempre visible |
| Presets | no tiene | 7 presets built-in (Today, Yesterday, This week…) |

---

## Sitios de uso en el backoffice

### Migrados ✅
- **`ticket-log`** — CDK overlay con `TicketLogDateFilter`
- **`validations-actions`, `cash-reconcile`, `ticket-transactions`, `daily`, `tip-report`, `time-tracking`, `overnight`** — CDK overlay con `DateFilterButton` genérico

### Candidatos para migración futura

**1. `custom-quick-filter`** — `/src/app/shared/components/custom-quick-filter/`
- Standalone, abre como `MatDialog`
- FormGroup: solo `period: FormControl<Date[]>`
- Más simple — candidato para una primera prueba limpia

### Otros sitios (menor prioridad)
- Reportes: `daily-report-detail/custom-period-dialog`, `time-tracking/update-shift`, etc.
- COT reports (~9 componentes bajo `/src/app/shared/components/cot-*`)
- `employee-form` (solo una fecha, no range)
- `overnight-rate-form`, `scheduled-rate` — solo timers → **no aplica** para esta migración

---

## Conversión de tipos

```ts
// DateTime → string backoffice
const startStr = event.range.start.toFormat("yyyy-MM-dd'T'HH:mm:00.000000")
const endStr   = event.range.end.toFormat("yyyy-MM-dd'T'HH:mm:59.999999")

// string ISO → DateRange (para hidratar value input)
import { DateTime } from 'luxon'
const value: DateRange = {
  start: DateTime.fromISO(existingStartStr),
  end:   DateTime.fromISO(existingEndStr),
}

// Date nativo → DateRange
const value: DateRange = {
  start: DateTime.fromJSDate(existingStart),
  end:   DateTime.fromJSDate(existingEnd),
}
```

Luxon ya está instalado en el backoffice (`"luxon": "^3.5.0"`).

---

## Bugs en cp-date-range-picker — ✅ CORREGIDOS (2026-06-01)

1. ~~Header "start" hace `onClear()` en lugar de volver a fase start~~ → corregido: `(click)="goToPhase('start')"`
2. ~~Header "end" no tiene acción~~ → corregido: `(click)="goToPhase('end')"`

---

## Responsive — `cp-date-range-picker` en mobile (≤640px)

Añadido en `date-range-picker.component.scss` — `@media (max-width: 640px)`:
- `.drp__body` cambia a `flex-direction: column` (calendario arriba, presets abajo)
- `.drp__left`: `min-width: 0`, `border-right: none`, `border-bottom` como separador
- `.drp__right`: `width: 100%`, padding y gap reducidos
- `.drp__presets`: `display: grid; grid-template-columns: repeat(3, 1fr)` — 3 columnas en lugar de lista vertical
- `.drp__preset`: `white-space: normal; text-align: center`
- `.drp__actions`: `flex-direction: row`, sin `border-top`

---

## Notas de integración

- `cp-date-range-picker` es `inline-flex` — se renderiza con su tamaño natural (~480px ancho) en desktop
- En mobile el contenedor (`DateFilterButton`) le pasa `width: 100%` vía `OverlayConfig` y el picker se adapta al ancho del overlay
- Los tokens del picker usan `--cp-ui-*` prefix — requiere que `styles.css` del DS esté importado en `angular.json` (ya está)
- Import correcto: `import { DateRangePickerComponent } from '@corepark/corepark-ui'`
