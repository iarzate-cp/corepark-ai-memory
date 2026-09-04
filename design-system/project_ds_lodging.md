---
name: project_ds_lodging
description: "ds-lodging: rebuild de /settings/lodging sobre la lib — tabla con fila expandible, un diálogo para alta y edición, y los cuatro defectos del clásico que mueren con él"
metadata:
  node_type: memory
  type: project
  modified: 2026-09-04
---

Hecho el **2026-09-04** en `feature/design-toggle`, commit `67cf11ff`, ya en `feature/staging`. Ruta duplicada con `newDesignGuard`; el módulo clásico intacto.

**Sin verificar en pantalla.**

## La forma

Misma regla de cardinalidad que `ds-rates`: una ubicación tiene muchos partners, con buscador y paginador, así que las unidades son **fila expandible**, no diálogo. Eso absorbió `units-detail`.

Los cinco componentes del clásico (página, container, tabla, y el par add/update) quedan en una página, una celda y **un** diálogo: `add-lodging-dialog` y `update-lodging-dialog` eran 154 y 121 líneas que solo se diferenciaban en el selector de partner, así que `DIALOG_DATA.configuration` a `null` o a un partner es lo que los separa.

**Un detalle del dominio que hay que entender antes de tocar esto:** la API devuelve *todos* los partners de la ubicación, y `units.length` es lo que distingue una configuración de una opción disponible. `lodgings$` (>0) son las filas de la tabla; `lodgingOptions$` (===0) son los partners a los que todavía se puede añadir configuración. Es la forma del endpoint, no un invento de la página vieja.

## Cuatro defectos del clásico que mueren con él

1. **El resumen se pedía dos veces.** La página corría `getLodgingSummary` y `lodging-table` declaraba su propio observable idéntico y lo volvía a correr tras cada diálogo.
2. **`lodging-container` existía solo para elegir entre tabla y `<alert>`.** `cp-table` tiene estado vacío, así que es un mensaje, no una rama. Los dos vacíos siguen distinguidos: sin partners vs. todos configurados.
3. **`ngOnChanges` + `detectChanges()`** empujando un `MatTableDataSource` nuevo.
4. **Dos suscripciones sin teardown** (`maxWidth$` en página y tabla).

## Decisiones

- **`mat-autocomplete` → `cp-select`.** El clásico llevaba `combineLatest` + `startWith('')` + `#filterOptions` + `displayWith` + botón de limpiar + `autocompleteValidator` guardando contra que el control se quedara con el texto crudo. `cp-select` es un combobox con buscador propio, así que todo eso es del componente y **el control guarda un `partnerId`**, no el objeto.
- **`dsLodgingForm` es nueva al lado de `lodgingForm`**, no un cambio.
- **`ds-lodging` no escribe en `LodgingState`.** Nada más lo lee: los partners disponibles se pasan por `DIALOG_DATA`, como el `TagDialogData.cars` de [[feedback_tag_dialog_cars]].
- **Chips estáticas, no `cp-pill-group`**, para las unidades: los items de ese componente son botones que emiten selección, y prometería una interacción que no existe.
- **i18n completo.** Todas las claves `LODGING.*` ya existían para el clásico; `core/i18n/lodging-i18n.ts` solo las tipa. Los `SUCCESS` se reusan tal cual como títulos de toast.

## El que se me escapó y hubo que devolver

**El foco automático en la unidad recién añadida.** El clásico lo hacía en `units-form-array` con `unitInputs.changes.pipe(debounceTime(100))`. Yo implementé el scroll y **escribí en un comentario que el foco también estaba resuelto**, cuando no lo estaba. Con cinco o seis unidades eso es un clic por unidad.

Restaurado con **`afterNextRender`**, que *es* el render: nada que adivinar y nada que desmontar. **No como `effect` sobre `viewChildren`** — eso dispararía también en el primer render y le robaría el foco al selector de partner al abrir el diálogo. El `changes` del clásico no emite en el render inicial, y ese detalle es el que hay que conservar.

De paso: había dejado **dos botones «Add unit»**, el del pie (`extra-action`) y otro dentro de `ds-unit-fields`. Se quedó el de dentro, pegado a los campos, como «Add tier» en `ds-tier-fields`.

Ver [[project_material_to_corepark_ui]], [[project_ds_rates]] y [[feedback_redesign_solo_new_design]].
