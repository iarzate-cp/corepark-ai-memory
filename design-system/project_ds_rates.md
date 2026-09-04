---
name: project_ds_rates
description: "ds-rates: el rebuild completo de rates sobre la lib — cómo quedó, los defectos del clásico que muere con él, y los cuatro fallos del rebuild ya corregidos"
metadata:
  node_type: memory
  type: project
  modified: 2026-09-04
---

Rama `feature/design-toggle` de `~/Dev/frontend-backoffice`, **mergeada a `feature/staging` el 2026-09-04**. Ruta duplicada con `newDesignGuard`; `RatesComponent` clásico intacto.

## Los fallos del rebuild, corregidos el 2026-09-04

Israel el 2026-09-03 dijo «hay muchas cosas que se rompieron» sin lista. Los fue soltando de uno en uno; salieron todos del **diálogo de tarifa variable** y del **overnight**.

1. **La cadena de tramos solo corría al añadir o quitar.** Teclear el «To» nunca rellenaba el «From» del tramo siguiente — el clásico lo propagaba en vivo con `(ngModelChange)`. Ahora corre desde el `valueChanges` del `FormArray`, que cubre las tres vías.
2. **El «Max» del último tramo quedaba editable y `required`.** El clásico lo deshabilita y lo vacía. Enabled hacía que **el Confirm naciera muerto al abrir una tarifa existente**, y al crear exigía un valor que luego se descartaba. Con eso se fue también un `clearValidators()` que dejaba sin `required` al tramo que dejaba de ser el último.
3. **`#toMinutes` partía por `':'`.** Con `dropSpecialCharacters` activo la máscara se come el separador y `'2:30'` llega al control como `'230'`: **13 800 minutos en vez de 150**. La conversión vive ahora en `@utils/parked-time`, que lee las dos formas.
4. **El overnight mandaba un solo campo de corte.** La API guarda `businessDayCutTime`, `checkInCutTime` y `cutTime` por separado y rechazaba con `INVALID-RATE-CONFIG-OVERNIGHT-06`, «must inform the cut times». El clásico manda los tres con el mismo valor.

**Lo que dejó pasar el 4: `as never`.** `NewOvernightRate` ya declaraba los tres como `Required`, pero el payload entraba al servicio con ese cast y tiraba la única comprobación que había. Quitado de los tres diálogos que lo tenían (variable, temporary, overnight); ya no queda ninguno en `ds-rates`, y el build en verde **sin casts** es la prueba de que los payloads cuadran de verdad.

**Layout:** la fila de tramos usaba `1fr` pelado. El `min-width: auto` implícito de los items de grid impide que la pista baje del ancho intrínseco del `<input>` (~180 px), así que tres campos más el botón desbordaban el diálogo con scroll horizontal. El clásico ya usaba `minmax(0, 1fr)` justo por esto.

Los dos defectos visuales del 2026-09-03, también arreglados:
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

- **Los uuid de cliente** del formulario de comps: solo eran claves de `track` y nunca se enviaban.

### `variable-time` se quitó y hubo que devolverlo

**Corregido el 2026-09-04.** Esta memoria justificaba haberlo quitado con «el stepper eran 48 clics para llegar a 24:00». **Es falso**: las horas tenían su propio contador (24 clics) y los dos inputs del panel eran escribibles. Israel lo notó como regresión — «cuando estoy en el input to, no hace nada y antes desplegaba algo».

Y no era un popover: su `:host` era `position: absolute; inset: 0` con fondo blanco, o sea **tapaba el diálogo entero**.

Reconstruido como **`ds-duration-panel`**, con dos cambios deliberados:
- **Overlay del CDK anclado al campo**, no una capa sobre todo el diálogo, así se ve el tramo que editas.
- **Abre desde un botón propio, no al enfocar**, así el campo enmascarado sigue siendo tecleable. El panel es una alternativa, no un peaje.

**No sirve `cp-time-field`** de la lib: es reloj de pared con AM/PM, y esto es una **duración** donde 26:00 es legítimo — el mismo razonamiento que ya está escrito para `ds-stay-time-fields`.

Los contadores guardan **strings, no números**: ngx-mask escribe de vuelta lo que pintó, así que el `FormControl<number>` del clásico con `value + 1` concatena `'2'` en `'21'` en vez de contar a 3.

## Decisiones que conviene no revisitar

- **La duración media de estancia son dos campos de texto con máscara**, no un campo de hora: es una **duración**, y 26 horas es una estancia media legítima que ningún reloj expresa.
- **El precio no es ordenable** en la tabla: el valor es un string formateado, así que ordenaría `$9.00` después de `$12.00`. El filtro sí matchea sobre él.
- **`dsScheduledRateGroup`, `dsFlatRateForm`, `dsVariableRateForm`, `dsOvernightRateForm` y `dsTemporaryRateForm`** son factorías nuevas **al lado** de las clásicas, no cambios: los tipos clásicos fijan `schedule` con `Date` y `period` con `Date[]`, y los diálogos clásicos siguen en ellos.

## Infraestructura que salió de aquí y sirve a todo

- **`core/services/notifier-service.ts`** — la fachada del toast del DS. Absorbe el mapa de códigos de error del backend, el mapeo status→severidad (4xx = warning, el resto error) y las duraciones (3500 éxito / 4500 error). Ver [[reference_notification_service]].
- **`core/enums/api-error-code.ts`** — los 12 códigos del backend, que eran strings mágicos.
- **`core/i18n/rates-i18n.ts`** y **`general-services-i18n.ts`** + los namespaces `RATES` y `GENERAL_SERVICES.ERROR` en `en`/`es`.
- **`core/utils/rate-detail-fields.ts`** — la única parte pura de rates, con **34 tests** que **no pueden correr**: el BO no tiene runner instalado (ni vitest, ni karma, ni jasmine), y ya hay 10 `.spec.ts` en la misma situación.
- **`core/utils/parked-time.ts`** (2026-09-04) — `parkedTimeToMinutes` / `minutesToParkedTime`, las dos formas del valor enmascarado. Lo usan el diálogo y el panel.

Ver [[project_material_to_corepark_ui]], [[project_bo_module_migration]], [[project_ds_lodging]] y [[project_ds_spot_configuration]].
