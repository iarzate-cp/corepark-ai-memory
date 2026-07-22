---
name: corepark-ui — Component & Directive API Surface
description: Selector, inputs, outputs and key behavior of every component/directive in the library
type: project
originSessionId: e29b3d2a-0295-4627-818f-8e980d288b27
---
## Entry point: `src/public-api.ts`

All exports flow through two barrels:
- `lib/tokens` → JS token constants
- `lib/components` → all components, directives, services
- `lib/directives`, `lib/pipes`, `lib/utils` → currently empty stubs

---

## Directives (standalone, no template)

### ButtonDirective — `[cpButton]`
```ts
variant: 'primary' | 'secondary' | 'text' = 'primary'
size:    'md' | 'sm' | 'xs'               = 'md'
iconOnly: boolean                          = false
```
- Applies host classes `cp-btn cp-btn--{variant} cp-btn--{size}` (+ `cp-btn--icon-only`)
- Styles in `button-directive.scss`

### InputDirective — `input[cpInput], textarea[cpInput]`
- No inputs — applies transparent bg, no border, flex-1
- Designed to live inside `<cp-form-field>`

### RippleDirective — `[cpRipple]`
- No inputs
- Material-style ripple on click; respects `prefers-reduced-motion`
- Adds `.cp-ripple-host` to host, injects `.cp-ripple` spans on click

### TooltipDirective — `[cpTooltip]`
```ts
cpTooltip:         string (required)
cpTooltipPosition: 'top' | 'bottom' | 'left' | 'right' = 'bottom'
```
- Uses CDK Overlay; shows `TooltipComponent` on mouseenter/focus
- 160ms leave animation

---

## Form Controls (ControlValueAccessor)

### SelectComponent — `cp-select`
```ts
label:       string         = ''
options:     SelectOption[] = []
multiple:    boolean        = false
placeholder: string         = 'Buscar...'
disabled:    boolean        = false
error:       string         = ''
hint:        string         = ''
size:        'sm'|'md'|'lg' = 'md'
```
`SelectOption = { value: unknown; label: string; disabled?: boolean }`
- CDK Overlay panel; keyboard nav (↑↓ Enter Esc); search filter; multi-select chips

### CheckboxComponent — `cp-checkbox`
```ts
checked:  boolean (model, two-way)
disabled: boolean = false
label:    string  = ''
```

### SwitchComponent — `cp-switch`
```ts
checked:  boolean (model, two-way)
disabled: boolean = false
variant:  'default' | 'check' = 'default'
```

### RadioGroupComponent — `cp-radio-group`
```ts
value:    unknown (model, two-way)
disabled: boolean = false
```
### RadioOptionComponent — `cp-radio-option`
```ts
value: unknown (required)
label: string  (required)
```
- Options self-register into group via `CP_RADIO_GROUP` injection token

---

## Layout / Navigation

### ShellLayoutComponent — `cp-shell-layout`
- No inputs — two-column grid (auto | 1fr), full viewport height

### SidebarComponent — `cp-sidebar`
```ts
items:        NavItem[] (required)
bottomItems:  NavItem[] = []
user:         SidebarUser | null = null
sectionLabel: string = ''
collapsed:    boolean (model)
// output:
itemClick: EventEmitter<NavItem>
```
`NavItem = { label, icon?, route?, children? }`
`SidebarUser = { name, email }`
- Router-aware active state; collapsible; nested groups with animation

### BreadcrumbComponent — `cp-breadcrumb`
```ts
items: BreadcrumbItem[] (required)
```
`BreadcrumbItem = { label, route?, icon? }`

### TabGroupComponent — `cp-tab-group`
```ts
selectedIndex: number = 0
tabChange:     EventEmitter<number>
```
### TabComponent — `cp-tab`
```ts
label:    string  (required)
disabled: boolean = false
```
- Animated indicator bar slides between tabs

---

## Overlays / Portals

### DialogService
```ts
open<R, D>(component: Type<unknown>, config?: DialogConfig<D>): DialogRef<R>
```
`DialogConfig<D> = { data?, width?, maxWidth?, disableClose? }`
`DialogRef<R> = { close(result?), beforeClosed(), afterClosed() }`
- Injection tokens: `DIALOG_DATA`, `DIALOG_CONFIG`
- Backdrop blur + scale/slide animation (160ms); ESC / backdrop click closes (unless `disableClose`)
- Container (`cp-dialog-container`) handles: built-in X close button, responsive sizing (`width: 28rem; max-width: min(calc(100vw - 2rem), 40rem)`), z-index, body portal
- `config.width` / `config.maxWidth` override the panel defaults via inline style

**Usage in opener:**
```ts
readonly #dialog = inject(DialogService)

openMyDialog(data: MyData): void {
  this.#dialog.open(MyDialogComponent, { data })
}
```

**Usage in dialog content component:**
```ts
readonly #data = inject<MyData>(DIALOG_DATA)
// DialogRef optional — container's X button already closes
readonly #ref = inject<DialogRef>(DialogRef) // only if you need programmatic close
```

**Dialog content template pattern** (no wrapper needed — container provides X button):
```html
<div class="my-dialog">
  <h2 class="my-dialog__title">{{ title }}</h2>
  <!-- content -->
</div>
```
```scss
.my-dialog { display: flex; flex-direction: column; gap: 16px; padding: 24px; }
.my-dialog__title { font-size: 16px; font-weight: 600; color: var(--color-text-950); margin: 0; }
```

**⚠️ `wrapper-dialog` is NOT compatible** — it uses `mat-dialog-close` (MatDialog dependency). Do not use `wrapper-dialog` inside a `DialogService`-opened component.

### NotificationService
```ts
show(config: NotificationConfig): void
dismiss(id: string): void
setPosition(position: NotificationPosition): void
```
`NotificationConfig = { type?, title (required), description?, duration? }`
`NotificationPosition = 'top-right'|'top-left'|'top-center'|'bottom-right'|'bottom-left'|'bottom-center'`
`NotificationType = 'info' | 'success' | 'error' | 'warning'` — ⚠️ usa `'error'`, no `'danger'`
- `providedIn: 'root'` con factory por defecto (posición `bottom-right`) — no requiere setup en `app.config`
- `provideNotifications(config)` es opcional, solo para cambiar la posición por defecto
- Estilos globales en `styles.scss` vía BEM (migrado de Tailwind en 2026-05-19)
- Injection tokens: `CP_NOTIFICATION_ITEMS`, `CP_NOTIFICATION_DISMISS`, `CP_NOTIFICATION_POSITION`
- Reemplaza `MatSnackBar` — inyectar y llamar `show()` directamente, sin imports en `imports: []`

### TooltipComponent — `cp-tooltip`
- Rendered by `TooltipDirective`; not used directly

---

## Data Display

### CpTableComponent — `cp-table`
```ts
columns:         TableColumn[] (required)
rows:            unknown[]     = []
pageSize:        number        = 25
pageSizeOptions: number[]      = [10,25,50,100]
collapsible:     boolean       = false
filterable:      boolean       = true
serverSide:      boolean       = false
loading:         boolean       = false
emptyMessage:    string        = 'No hay datos'
// outputs:
sortChange:   EventEmitter<TableSort>
pageChange:   EventEmitter<TablePage>
filterChange: EventEmitter<string>
```
`TableColumn = { key, label, sortable?, align?, width?, hideBelow? }`
- Custom cell templates via `CpCellDef` directive: `<ng-template cpCell="key" let-row>`
- Custom expand template via `CpExpandDef` directive: `<ng-template cpExpand let-row>`
- Expandable rows with animation; client-side or server-side mode

### StatCardComponent — `cp-stat-card`
```ts
label:     string      (required)
value:     string      (required)
variant:   'default'|'icon'|'progress'|'sparkline'|'bar'|'goal' = 'default'
trend:     number|null = null
icon:      TablerIcon|null = null
iconColor: 'primary'|'success'|'danger'|'warning'|'info' = 'primary'
progress:  number      = 0
data:      number[]    = []
target:    string      = ''
```
- SVG sparkline and bar visualizations inline

### BadgeComponent — `cp-badge`
```ts
variant: 'success'|'danger'|'warning'|'info'|'default' = 'default'
dot:     boolean = false
size:    'sm'|'md'                                       = 'md'
```

---

## Charts (SVG, no third-party charting lib)

All charts live in `lib/components/charts/`.

### LineChartComponent — `cp-line-chart`
```ts
series:     ChartSeries[] (required)
labels:     string[]      (required)
height:     number        = 240
showArea:   boolean       = true
yTickCount: number        = 5
```

### BarChartComponent — `cp-bar-chart`
```ts
series:     ChartSeries[] (required)
labels:     string[]      (required)
height:     number        = 240
groupPad:   number        = 0.3
yTickCount: number        = 5
```

### StackedBarChartComponent — `cp-stacked-bar-chart`
```ts
bars:        StackedBar[]  (required)   // { label, heightPct, segments: { flexPct, color }[] }
yTicks:      number[]      (required)
yAxisLabel:  string        = ''
```
- Layout CSS puro (no SVG): flex column con `.sbc__plot` + `.sbc__x-row`
- Grid lines: `.sbc__grid` con `position: absolute; bottom: X%`; solo se renderizan para `tick > 0` (el tick=0 causaría doble línea con el border-top del x-row)
- Eje X: `border-top: 1px solid` en `.sbc__x-row` (NO en `.sbc__plot`) — evita overlap visual

### DonutChartComponent — `cp-donut-chart`
```ts
segments:    DonutSegment[] (required)
size:        number         = 200
thickness:   number         = 36
centerLabel: string         = ''
centerValue: string         = ''
```

**⚠️ Fix doble línea (2026-06-09):** el grid-line SVG en tick=0 coincide con la `<line class="baseline">` — siempre usar `@if (tick > 0)` en el loop de y-axis ticks para no renderizar el grid line en la base.

**Shared types:**
```ts
ChartSeries  = { name, data: readonly number[], color? }
DonutSegment = { label, value, color? }
```

**Chart utilities (`chart-utils.ts`):**
`linearScale`, `niceLinearTicks`, `buildLinePath`, `buildAreaPath`, `polarToCartesian`, `donutArcPath`, `formatAxisValue`
- Charts use `ResizeObserver` for responsive width
- Hover tooltips built-in

---

## Feedback / Status

### AlertComponent — `cp-alert`
```ts
variant:      'info' | 'success' | 'danger' | 'warning' = 'info'
title:        string  = ''
dismissible:  boolean = false
dismissLabel: string  = 'Dismiss'   // pass translated string from consumer
// output:
dismissed: EventEmitter<void>
```
- Inline static alert — distinto a `cp-notification` que es toast/servicio
- BEM: `.cp-alert`, `.cp-alert--{variant}`, `.cp-alert__icon`, `.cp-alert__body`, `.cp-alert__title`, `.cp-alert__dismiss`
- Usa `--color-{variant}` + `--color-{variant}-alpha` del token system
- Tabler icon en `__icon`; border-left 3px solid currentColor
- `dismissLabel` es un input para que el consumidor pase el texto traducido (i18n)

### DialogContentComponent — `cp-dialog-content`
```ts
headlining:   string  (required)
exitLabel:    string  = 'Exit'      // pass translated string from consumer
disabled:     boolean = false
extraActions: boolean = false
paddingless:  boolean = false
// output:
exitClicked: EventEmitter<void>
```
**Public methods (para control externo vía `@ViewChild`):**
```ts
scrollBodyTo(options: ScrollToOptions): void
scrollBodyToBottom(): void
setBodyOverflow(overflow: string): void
```
**ng-content slots:**
- default → body content
- `[action]` → botones de acción en footer (derecha)
- `[extra-action]` → acciones adicionales en footer (izquierda, cuando `extraActions=true`)

**Cierre dual:** emite `exitClicked` Y llama `this.#dialogRef?.close()` (DialogService). Consumidores con MatDialog escuchan `(exitClicked)` y llaman `this.#matDialogRef?.close()`.

**En el backoffice**, `wrapper-dialog` y `dialog-wrapper` delegan internamente a `cp-dialog-content` — sus ~60 consumidores no cambiaron.

---

## Misc Components

### AvatarComponent — `cp-avatar`
```ts
name:  string      (required)
color: string      = 'var(--cp-ui-color-primary)'  // CSS color/variable
size:  'sm'|'md'|'lg' = 'md'
```
- Auto-extrae iniciales: primeras letras de las primeras 2 palabras del `name` input
- `[style.background]` bound al host; texto siempre `#fff`
- Tamaños: sm=24px, md=32px, lg=40px

### PillGroupComponent — `cp-pill-group`
```ts
items:          PillItem[] (required)
selectedId:     string     = ''
// output:
selectedChange: string
```
`PillItem = { id: string; label: string; dot?: string }` — `dot` es un color CSS opcional (muestra círculo de color por item)
- Control segmentado (single-select), diseñado para toggles de métricas / filtros
- Host es el contenedor; fondo `--cp-ui-color-bg-80`; pill activo `--cp-ui-color-primary`

### ProgressBarComponent — `cp-progress-bar`
```ts
value:     number                                               (required)
variant:   'auto'|'success'|'warning'|'danger'|'primary'|'info' = 'auto'
size:      'sm'|'md'                                             = 'md'
showLabel: boolean                                               = false
```
- `variant='auto'`: verde < 70%, ámbar 70–89%, rojo ≥ 90%
- Label opcional (porcentaje) a la derecha del track
- `#resolvedVariant` es `#private` (solo lo usa `fillClass`); `clampedValue`, `hostClass`, `fillClass` son `protected` (accedidos desde template)

---

### FormFieldComponent — `cp-form-field`
```ts
label:   string              = ''
error:   string              = ''
hint:    string              = ''
variant: 'text'|'textarea'   = 'text'
size:    'sm'|'md'|'lg'      = 'md'
disabled: boolean            = false
```
- Floating label with Material-style notch outline
- States: `.is-floating`, `.is-focused`, `.is-error`, `.is-disabled`
- Project prefix/suffix icons via ng-content

### GridComponent — `cp-grid`
```ts
columns: number|string          = 1
rows:    number|string|undefined = undefined
gap:     string                  = '1rem'
custom:  string|undefined        = undefined
```

### CoreparkIsotypeComponent — `cp-isotype`
```ts
filled:     boolean = true
colorFill:  string  = 'var(--color-text-950)'
height:     string  = '3rem'
```

### DateRangePickerComponent — `cp-date-range-picker`
```ts
value:          DateRange|null  = null
options:        PickerOption[]  = []     // footer checkboxes; hidden when array is empty
showOvernight:  boolean         = false  // renders "Overnight" checkbox in footer
overnightValue: boolean         = false  // initial/reset value for overnight checkbox
// outputs:
apply: OutputEmitter<PickerApplyEvent>
clear: OutputEmitter<void>
```
**Types (exported from library index):**
```ts
DateRange        = { start: DateTime, end: DateTime }          // Luxon DateTime
PickerOption     = { readonly id: string; readonly label: string; readonly checked: boolean }
PickerApplyEvent = { readonly range: DateRange; readonly options: ReadonlyArray<PickerOption>; readonly overnight: boolean }
```
- Preset shortcuts (today/yesterday/this-week/last-week/this-month/last-month/this-year)
- Time picker (12h AM/PM); 6-week calendar grid; range hover highlight
- `options` footer: renders `cp-checkbox` per option, state managed internally via `#optionStates` signal
- Footer visible when `options.length > 0 || showOvernight`
- `overnight` is always present in `PickerApplyEvent`; only meaningful when `showOvernight=true`
- On clear: resets dates, resets option states AND resets overnight to `overnightValue`
