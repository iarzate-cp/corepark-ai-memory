---
name: project_ds_rates
description: "ds-rates: el rebuild completo de rates sobre la lib — cómo quedó, los defectos del clásico que muere con él, y que está roto sin lista todavía"
metadata:
  node_type: memory
  type: project
  modified: 2026-09-03
---

Rama `feature/design-toggle` de `~/Dev/frontend-backoffice`, **sin empujar**. Ruta duplicada con `newDesignGuard`; `RatesComponent` clásico intacto.

## ⚠️ Está roto

Israel, el **2026-09-03**: «no hemos terminado con rates, hay muchas cosas que se rompieron». **No hay lista todavía.** Antes de tocar nada más aquí, pedirle qué falla — es el único que lo ha visto en pantalla.

Los dos defectos visuales que sí reportó y están arreglados:
1. **Cabecera duplicada.** Puse `data: { ownHeader: false }` *y* pinté mi propio `cp-page-header`. La bandera significa lo contrario de lo que creí: `app-layout.component.html` es `@if (heading() && !ownHeader())`, así que **`false` = la pinta el shell**. Sin bandera (default `true`) la pinta la página.
2. **Heading invertido.** Puse la ubicación como título y «Rates» como subtítulo. Sale de la ruta, vía `TranslatedTitleStrategy.heading` / `.subheading`, y la ubicación **no** va en el texto — ya la nombra el `page-meta-row`.

## Cómo quedó

**Cero `@angular/material`.** Tabla en `cp-table`, cabecera con `cpButton` + `cp-menu`, avisos por `NotifierService`, y **los 15 diálogos reconstruidos**.

**Diez componentes se volvieron cinco.** Cada tipo de tarifa tenía un create y un update con el formulario, el manejo de nombre duplicado y el submit duplicados línea por línea. `DIALOG_DATA` con una tarifa o `null` es lo que distingue el trabajo. Lo mismo con las secuencias (add y edit) y con el formulario de estación.

**El detalle es fila expandible, no diálogo** — por la regla de cardinalidad: muchas tarifas, con buscador y paginador. Eso absorbió `detail-rate-dialog`. Dentro de la fila va un `meta-strip`, no una tabla anidada, porque el detalle son pares etiqueta/valor.

**`disableClose: true` en todos.** `DialogConfig.disableClose` se lee una vez al abrir, al contrario que `MatDialogRef.disableClose` que el clásico volteaba a mano. Es más estricto a propósito: antes un clic en el backdrop a media petición cerraba el diálogo mientras la petición seguía, y **la tabla nunca se enteraba de que la tarifa se había creado**.

## Defectos del clásico que mueren con él

Todos siguen vivos en `RatesComponent`, que es el que ve el diseño clásico:

1. **El buscador está muerto.** `(ngModelChange)` sobre un input con `[formControl]`: sin `NgModel` no hay quien dispare `applyFilter()`, y no tiene otro call site.
2. **El sort también.** `MatSort` inyectado y `MatSortModule` importado, pero la tabla no tiene `matSort` ni ninguna columna es sort header.
3. **`dataSource` es un `computed` con efectos** que construye un `MatTableDataSource` nuevo por cambio y llama a `searcherControl.disable()` dentro de la derivación, así que el filtro activo se pierde cuando cambian las tarifas.
4. **`colspan="4"`** en la fila de vacío, sobre seis columnas.
5. **Tres fugas de suscripción** en `ngOnInit` sin `takeUntilDestroyed`, y una es por fila.
6. **`find` lineal por celda** para recuperar el `Rate`, dos veces por fila por ciclo de detección.
7. **`prices.at(-1)`** para el precio variable: es el último tramo, no el más barato. Y `prices.at(0).price` sin guarda revienta la tabla si llega vacío.
8. **El refetch estaba en los componentes de secuencia**, así que veinte filas llevaban veinte copias.

## Piezas que se quitaron, no se portaron

- **`variable-time`**, un popover de flechas arriba/abajo para poner horas y minutos. El campo tiene máscara, así que teclear `2:30` es toda la interacción; el stepper eran **48 clics** para llegar a 24:00.
- **El `(ngModelChange)` por fila** que encadenaba los tramos. Un método reescribe la cadena al añadir o quitar, así que un cambio en medio no deja hueco.
- **Los uuid de cliente** del formulario de comps: solo eran claves de `track` y nunca se enviaban.

## Decisiones que conviene no revisitar

- **La duración media de estancia son dos campos de texto con máscara**, no un campo de hora: es una **duración**, y 26 horas es una estancia media legítima que ningún reloj expresa.
- **El precio no es ordenable** en la tabla: el valor es un string formateado, así que ordenaría `$9.00` después de `$12.00`. El filtro sí matchea sobre él.
- **`dsScheduledRateGroup`, `dsFlatRateForm`, `dsVariableRateForm`, `dsOvernightRateForm` y `dsTemporaryRateForm`** son factorías nuevas **al lado** de las clásicas, no cambios: los tipos clásicos fijan `schedule` con `Date` y `period` con `Date[]`, y los diálogos clásicos siguen en ellos.

## Infraestructura que salió de aquí y sirve a todo

- **`core/services/notifier-service.ts`** — la fachada del toast del DS. Absorbe el mapa de códigos de error del backend, el mapeo status→severidad (4xx = warning, el resto error) y las duraciones (3500 éxito / 4500 error). Ver [[reference_notification_service]].
- **`core/enums/api-error-code.ts`** — los 12 códigos del backend, que eran strings mágicos.
- **`core/i18n/rates-i18n.ts`** y **`general-services-i18n.ts`** + los namespaces `RATES` y `GENERAL_SERVICES.ERROR` en `en`/`es`.
- **`core/utils/rate-detail-fields.ts`** — la única parte pura de rates, con **34 tests** que **no pueden correr**: el BO no tiene runner instalado (ni vitest, ni karma, ni jasmine), y ya hay 10 `.spec.ts` en la misma situación.

Ver [[project_material_to_corepark_ui]] y [[project_bo_module_migration]].
