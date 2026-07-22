---
name: DateFilterButton — componente genérico de filtro de fecha
description: Componente shared del backoffice que encapsula CDK overlay + cp-date-range-picker; reemplaza el patrón MatDialog.open(FullDateFilterComponent) en páginas de reportes
type: project
originSessionId: f1d14aab-a36f-4dc1-b961-fde3c10a1821
---
## Ubicación

`/Users/israel/Dev/frontend-backoffice/src/app/shared/components/date-filter-button/`

- `date-filter-button.component.ts`
- `date-filter-button.component.html`
- `date-filter-button.component.scss`
- `index.ts` → `export { DateFilterButton }`

Import en consumidores: `import { DateFilterButton } from '@components/date-filter-button'`

---

## API

```ts
readonly showOvernight = input(false)  // muestra toggle overnight; setea isTicketTransactionsOvernight en apply
readonly showOptions   = input(false)  // muestra los 3 checkboxes (Inventory / Departed / Temp checkout)
```

Cuando `showOptions = false` (default), el picker recibe `options: []` — los checkboxes no aparecen y los valores del estado permanecen como estaban.

---

## Comportamiento

- Botón con `actioner-raised` + ícono filtro; muestra el label de fecha del `FullDateFilterState.label$`
- Al hacer click abre overlay vía CDK `Overlay` service (imperativo) + `TemplatePortal` sobre `#panelTpl`
- `BreakpointObserver` detecta `(max-width: 640px)` como signal `#isMobile`
- `#openOverlay()` llama `#mobileConfig()` o `#desktopConfig()` según `#isMobile()` — sin ternarios en los configs

**Desktop** (`#desktopConfig()`):
- `FlexibleConnectedPositionStrategy` anclado a `#triggerBtn`; posiciones: bottom-right, fallback top-right
- `cdk-overlay-transparent-backdrop`
- Animación: `translateY(-6px) scale(0.97)` → normal (160ms)

**Mobile** (`#mobileConfig()`):
- `GlobalPositionStrategy` sin posición explícita — el posicionamiento lo hace CSS puro
- `cdk-overlay-dark-backdrop`
- `.panel` tiene `position: fixed; top: 50%; left: 50%; width: calc(100dvw - 2rem); max-height: calc(100dvh - 2rem); overflow-y: auto`
- Animación con transform combinado: `translate(-50%, -50%) scale(0.95)` → `translate(-50%, -50%)` — el translate está en los keyframes, no en el elemento, para no conflictuar

**Por qué `position: fixed` en `.panel` y no en el overlay config:**
`GlobalPositionStrategy` calcula `left` leyendo `getBoundingClientRect().width` del pane. Si `cp-date-range-picker` tiene `display: inline-flex` host y el ancho natural (~480px) > viewport, el cálculo da negativo y CDK lo clipa a `left: 0`. El enfoque CSS puro evita esta dependencia completamente.

**Cierre:**
- `isLeaving` signal → `#close()` → `isLeaving.set(true)` → setTimeout(160ms) → `#destroyOverlay()`
- `onApply()` llama `#destroyOverlay()` directamente (sin animación de salida)
- Escape y backdrop click cierran con animación

**⚠️ Gotcha: `viewChild.required` no puede usarse en campos ES private (`#`)**
Angular lanza NG1053. Los campos deben ser `protected readonly triggerBtn` y `protected readonly panelTpl`.

---

## TicketLogDateFilter — mismo patrón

`pages/main/reports/ticket-log/ui/ticket-log-date-filter/` tiene exactamente la misma implementación pero sin inputs `showOvernight`/`showOptions` (siempre muestra options con los 3 checkboxes). Fue migrado de `CdkConnectedOverlay` → `Overlay` service en la misma sesión.

---

## Uso en páginas

| Componente | Página | Input |
|---|---|---|
| `DateFilterButton` | `validations-actions` | (ninguno) |
| `DateFilterButton` | `cash-reconcile` | (ninguno) |
| `DateFilterButton` | `ticket-transactions` | `[showOvernight]="true"` |
| `DateFilterButton` | `daily` | (ninguno) |
| `DateFilterButton` | `tip-report` | (ninguno) |
| `DateFilterButton` | `time-tracking` | (ninguno) |
| `DateFilterButton` | `overnight` | `[showOptions]="true"` |
| `TicketLogDateFilter` | `ticket-log` | (siempre muestra options) |

---

## Lo que se eliminó al migrar

Cada página que migró de `MatDialog` → `DateFilterButton` eliminó:
- `MatDialog` + `FullDateFilterComponent` import
- `FullDateFilterLayoutState` inject + `#maxWidth` computed
- Constructor `ngOnInit()` con `isDateRange.next()` call
- El botón filtro inline con el `openFilterDialog()` handler

`FullDateFilterState` se MANTIENE en todas las páginas que lo necesitan para sus params de HTTP.
