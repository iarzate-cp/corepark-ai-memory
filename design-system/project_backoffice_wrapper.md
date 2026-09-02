---
name: project_backoffice_wrapper
description: "el wrapper de página del BO es único; absorbió rp-wrapper, pin-codes-wrapper y payments-wrapper con context + slot [meta]"
metadata: 
  node_type: memory
  type: project
  originSessionId: aea67aa5-d32a-4d47-b014-93e055e5f1b9
  modified: 2026-09-01T23:05:36.221Z
---

Homologado el **2026-08-28**. `shared/components/wrapper/` es **el único wrapper de página** del backoffice.

Es un shim de app sobre `cp-page-header` ([[project_cp_page_header]]): la lib pone la presentación, el shim el estado. Las páginas siguen proyectando en `[actions]` y `[content]` sin tocarse.

> **Adelgazó el 2026-09-01.** La fila de meta —breadcrumb o selector de parking, según `data.meta`— salió a **`page-meta-row`**, porque el shell tiene que pintar la misma cuando una página deja de usar el wrapper. Con ella se fueron del wrapper `MatDialog`, `MatRipple`, `Breadcrumb`, `CatalogueState` y el cálculo del ancho responsive del diálogo. Queda en heading + dos proyecciones.
>
> **Y ya no es el único que pinta cabecera:** bajo el diseño nuevo el shell la pinta desde la ruta, y el wrapper sirve a las páginas aún no reconstruidas. El interruptor es `data.ownHeader`, que **arranca en `true`** mientras queden 140 plantillas con `<wrapper>` — un shell que pintara por defecto les daría dos. Ver [[project_bo_module_migration]].

## Los dos escapes que absorbieron a los tres variantes

**`context`** — prefijo vivo que la ruta no puede conocer:

```html
<wrapper [context]="context()">   <!-- "Downtown Garage Partners | Request Points" -->
```

Era la razón de existir de `rp-wrapper`: `REQUEST_POINTS.HEADLINING` es `"{{ parkingLotName }} Partners"`, una **traducción con parámetro** alimentada por estado. No cabe en un `title` estático.

**`[meta]`** — sustituye lo que va bajo el título. **Sin comprobación de vacío**: la página que proyecta declara `data: { meta: 'none' }` y el `@switch` no pinta nada. La intención se declara en la ruta, como todo lo demás.

## Qué pasó con cada uno

| Antes | Ahora |
|---|---|
| `rp-wrapper` | `<wrapper [context]="context()">` + ruta `meta: 'breadcrumb'` |
| `pin-codes-wrapper` | `<wrapper [context]="'…EMPLOYEE_CENTER.LABEL' \| translate">` + `meta: 'breadcrumb'` |
| `payments-wrapper` | `<wrapper>` + `<ng-container meta><small>Global</small></ng-container>` + `meta: 'none'` |

**`payments-wrapper` no era un wrapper**: 6 líneas de cabecera y ~120 de carga de datos, un `@HostListener('window:storage')` para el OAuth de Square/Stripe y tres snackbars. Se partió: todo eso vive en **`PaymentsBootstrap`** (`@services/payments-bootstrap`), **provisto en la ruta `payments`** — una instancia para las dos pantallas, destruida al salir de la sección, que es lo que `takeUntilDestroyed` necesita para desenganchar el listener. El `@HostListener` pasó a `fromEvent`, que ya no exige estar en un componente.

## Código muerto que salió a la luz

1. **`pin-codes-wrapper.headlining$`** calculaba el nombre del empleado y su plantilla **nunca lo usaba**.
2. **El enlace "Go to Payments" nunca se renderizaba**: `isSpanVisible` era `!headlining.includes('Add Payment Gateway')` y el mapa solo producía `'Payments'` y `'Payments | Credentials'` — siempre true.
3. **`padding: 4rem` propio en los tres**: el doble inset que ya se había quitado del wrapper principal, todavía vivo en esas 4 páginas.

## NO tocar: la familia de diálogos

Decisión de Israel: `wrapper-dialog` (**259 usos**), `dialog-wrapper` (31), `dialog-wrapper-loader` (2), `ui-dialog-wrapper` (2) son **otro problema** y se migran de uno en uno, nunca por lote. Ver [[feedback_backoffice_dialog_wrappers]].

## Deuda que queda

- **`styleComponent` es inerte**: sigue aplicando la clase al host pero ya no hay CSS que la use (verificado: ningún estilo global las referencia). 4 call sites.
- El caso `ticket-transactions` ponía `width: 100%` a los *hijos* del contenedor de acciones; el DS lo pone al contenedor.
- **`page-label.component.ts` de commerce** es una tercera copia del árbol de rutas y solo lo usa el `main-layout` muerto.
