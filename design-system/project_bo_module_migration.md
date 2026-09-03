---
name: project_bo_module_migration
description: "backoffice: cómo se reconstruye un módulo sobre la lib bajo el diseño nuevo — patrón de ruta, qué se reutiliza, y las trampas que ya costaron un build"
metadata: 
  node_type: memory
  type: project
  originSessionId: 3c304b9d-9ff8-4ae9-aeb6-7e2f0a049f97
  modified: 2026-09-01T23:04:16.064Z
---

Empezado el **2026-09-01** en `~/Dev/frontend-backoffice`, rama `feature/design-toggle` (mergeada a `feature/staging` y empujada; app en `2026.9.1`, lib en `0.0.30` **del registro**).

## La regla que puso Israel

1. Bajo el diseño nuevo se muestra un **módulo completamente nuevo**.
2. **No se reescribe el que existe.**
3. Solo aplica cuando el diseño nuevo está activo.
4. **La forma es básicamente la misma** — no es rediseño, es reconstruir con piezas de la lib.

## El patrón: ruta declarada dos veces + `newDesignGuard`

`core/guards/design/design.guard.ts` → `CanMatchFn` sobre `DesignState.isNew()`.

```ts
{ path: 'x', canMatch: [newDesignGuard], loadComponent: …ds-x },
{ path: 'x',                             loadComponent: …x },   // fallback
```

Solo la entrada nueva lleva guard, así que la legacy es el fallback: un módulo aún no reconstruido tiene una entrada única y no se entera de nada. `isNew` ya incluye la elegibilidad por operador. `canMatch` corre **antes** de `loadComponent`, así que cada diseño descarga solo su chunk.

**Asimetría con el shell raíz, que usa `DesignShell` con `@if`:** ahí duplicar la entrada obligaba a duplicar 46 rutas hijas. Por módulo la duplicación es una entrada. Mismo criterio, distinto coste.

**Si el módulo absorbe una ruta hija** (partners→request-points, employee-center→pin-codes), la hija se declara con `canMatch: [newDesignGuard]` y `redirectTo` al padre — no se borra, porque el diseño clásico sigue enlazándola.

## La decisión que más se repite: qué reconstruir y qué reutilizar

**Un módulo nuevo puede ser parcial**, y eso es buena parte de lo que justifica la duplicación. Se reutiliza tal cual lo que es grande o compartido:

- `/locations`: los 4 diálogos y `daily-report-detail` (780 líneas, compartidas con otra página).
- `/locations/partners`: los 4 diálogos; `partner-form` solo son 197 líneas de Material más Google Places.
- `/employee-center`: los 4 diálogos de empleado y los 3 de pin code.
- `profiling`: los 7 diálogos.

**Contrato a respetar al reutilizar diálogos:** varios leen su objeto de un estado global (`PartnersState.partner`, `EmployeeCenterState.employee`). El módulo nuevo **tiene que seguir escribiéndolo**. Comprobar antes: los 4 de guest-profile **no** lo hacen —reciben todo por `MAT_DIALOG_DATA`— y eso permitió que el panel de coches fuera autónomo.

## La forma depende de la cardinalidad del padre

Partners y employee-center tenían **el mismo acoplamiento** —página hija + estado global + guard que expulsa— y **soluciones distintas**:

| Padre | Solución | Por qué |
|---|---|---|
| Partners (pocos) | **Una página** con selector de partner | La página se puede acotar a "el partner" |
| Empleados (muchos) | **Fila expandible** | Hay buscador, paginador y filtro; no se acota a "el empleado" |
| Guest profiles (muchos) | **Fila expandible** | Igual |

> **Regla:** el acoplamiento es el mismo; la forma la decide **cuántos elementos tiene el padre**.

**Y dentro de la fila expandida:** tabla anidada solo si el contenido *es* una tabla. Un vehículo son 7 campos con tags y acciones propias → `cp-table`. Un pin code son 3 datos y 2 botones → lista. No es cuestión de tamaño sino de qué es cada cosa.

## Reglas de composición que ya costaron un arreglo

- **`cp-page-header` es dueño del espacio que hay debajo** (`margin-block-end`). La página **no** debe añadir otro: un `:host { display:flex; gap }` lo suma y el header queda al doble de distancia. Host en `display: block`, y que solo espacien los bloques **posteriores**. Lo cazó Israel con el overlay de grid de DevTools — leyendo el CSS no se ve, las dos reglas son correctas por separado.
- **El shell pinta la cabecera de página** (flujo replicado de CMRS, commit `39be1c68`). `ownHeader` **arranca en `true`** mientras 140 plantillas usen `wrapper`; un módulo reconstruido se apunta con `data: { ownHeader: false }`. Al caer el último wrapper: invertir el default y borrar la bandera. Una página con **acciones** en la cabecera la pinta ella, porque una cabecera del shell no puede recibir contenido proyectado.
- **La fila de meta** (breadcrumb / selector de parking) vive en `page-meta-row`, compartida por el shell y el wrapper.
- **`meta-strip`** es el bloque de pares etiqueta/valor. Dos disposiciones: `row` recorta a una línea (los items del grid se estiran al más alto y un valor largo arrastra a todos) y `stack` envuelve (en columna no se estira nada, y recortar una dirección en 22rem no deja nada legible).
- **No usar chrome de tarjeta para datos de identidad** si al lado hay métricas: iguala el peso visual de referencia y cifras. Un filete fino basta.

## Trampas que costaron un build (o una captura)

1. **Backticks dentro de un template inline** — cortan el template literal. Un comentario HTML con `` `action` `` produjo NG1001 + cinco errores de TS apuntando al decorador. Me pasó **dos veces**. Ver [[feedback_angular_gotchas]].
2. **`toObservable()` exige contexto de inyección** — en un método privado revienta en runtime, no en build.
3. **`toSignal` se suscribe durante la construcción** — un `startWith` que lee un `input.required` lanza NG0950. Ver [[feedback_angular_gotchas]].
4. **`CurrencyPipe` inyectado va en `providers`, no en `imports`** (NG8113 si está en imports sin usarse en plantilla).
5. **`cp-menu-item[disabled]` hace `stopPropagation()`**, que no bloquea el `(click)` del consumidor — early return en cada handler.
6. **Selectores de proyección de `cp-dialog-content`:** default → cuerpo, **`[action]`** → pie junto al botón de salida, **`[extra-action]`** → solo si `extraActions()`. Un selector mal escrito **no da error, no proyecta**.
7. **Una entidad HTML dentro de una cadena de TS no se decodifica** — `{{ cond ? '&#9679;' : '…' }}` imprime el texto literal. En markup sí.
8. **Los estilos encapsulados no alcanzan el contenido proyectado.** Si el componente quiere colocarlo, tiene que envolver su propio slot.

## Lo que se le añadió a la lib por el camino

Cuatro veces la respuesta correcta fue **añadir a la lib, no parchear el consumidor**: `rowEmphasis`, `singleExpand`, el slot `[cpTableToolbar]` con `searchPlaceholder`, y dos arreglos de layout del toolbar. Detalle de API en [[project_corepark_ui_components]].

**Conviene mirar si falta un hook en la lib antes de escribir CSS propio en un módulo.**

## Bugs encontrados en el código clásico

- **partners:** `STORAGE_KEYS.PARTNER` se escribía y **nadie lo leía**, y se borraba justo al navegar. `rp-table` renderizaba desde el input mientras buscador y paginador manejaban un `MatTableDataSource` invisible — **ambos muertos**; mensajes de vacío invertidos.
- **employee-center:** `STORAGE_KEYS.EMPLOYEE` **se lee y nadie lo escribe** — la mitad opuesta del mismo mecanismo roto. `getEmployees()` pide `limitRows: 11520` y pagina en cliente aunque la respuesta trae `totalRows`.
- **profiling:** el "search by" mandaba solo `TAG` al servidor; los otros cinco caían en el filtro de Material, que busca en todas las columnas. "First name", "Last name" y "Phone" hacían lo mismo que "All fields", y **"Plate" y "VIN" no podían funcionar** (no están en la fila). El endpoint acepta los seis desde siempre.
- **`/locations`:** media query `(max-width: 1600px) and (min-width: 1920px)` — máximo bajo su propio mínimo, nunca casaba.
- **`PhoneNumberByCountryCodePipe`** declaraba `value: number` recibiendo `string`; nunca falló porque su cuerpo empieza con `toString()` y **todos los call sites eran plantillas**.

## Módulos hechos

| Módulo | Commit BO | Forma |
|---|---|---|
| `/` dashboard | `997b819a` | reconstruido |
| `/analytics/activity-by-rate-class` | `d9942424` | tabla transpuesta → `cp-table` |
| `/locations` | `22266419` | reconstruido, informe reutilizado |
| `/locations/partners` (+ request points) | `a2bb6f05` | una página con selector |
| `/employee-center` (+ pin codes) | `20086756` | fila expandible |
| `/guest-settings/profiling` | `b9cd3e5b` | fila expandible con tabla anidada |
| `/settings/rates` | `db94957e`…`55d2511c` | fila expandible; **cero Material**; ver [[project_ds_rates]] |
| `/settings/stations` | `d19e3f42` | contenedores + diálogos + snackbar |
| `/settings/tipping` | `0c633145` | contenedores + controles + snackbar; **la cabecera la pinta el shell** |

## El alcance no tiene que ser el módulo entero

Rates fue completo. **Stations y tipping fueron «contenedores, diálogos y snackbar»** por decisión de Israel, y salió mejor: módulos cortos, riesgo acotado, y el clásico intacto igual. No hay que reconstruirlo todo para sacar Material de un módulo.

## `ownHeader`: significa lo contrario de lo que parece

`app-layout.component.html` es `@if (heading() && !ownHeader())`. O sea:

- **sin bandera** (default `true`) → **la página pinta su cabecera**. Es lo que hace falta si hay un control arriba, porque una cabecera del shell no recibe contenido proyectado.
- **`data: { ownHeader: false }`** → **la pinta el shell**. Solo si la página no tiene acciones.

Me lo salté en `ds-rates`: puse la bandera *y* pinté la mía, y salieron las dos, cada una con su `page-meta-row`. `ds-tipping` es el primero que la usa de verdad.

Y el heading **sale de la ruta** vía `TranslatedTitleStrategy.heading` / `.subheading`. La ubicación **no** va en el texto: ya la nombra el selector de parking del `page-meta-row`.

## Sin verificar

**Nada se ha visto en pantalla más allá de lo que Israel revisó a mano**, y todo lo que revisó tenía algo — doble espaciado, tarjetas que se repetían, acciones apiladas, glifos perdidos, un selector desalineado. **El método que funciona es él mirando y yo buscando la causa; leyendo el CSS no salen.**

Sigue sin pulsar nadie el toggle del diseño. Ver [[project_design_toggle_backoffice]] y [[project_pending_work]].
