---
name: gotchas-de-angular-y-tooling-descubiertos-en-la-migraci-n-de-layouts
description: "proyección de contenido y @if, selectores de atributo que no fallan, pnpm minimumReleaseAge, publish:lib, orden de styles en angular.json"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-09-01T23:04:54.124Z
---

Encontrados el 2026-08-26 migrando layouts. Todos costaron tiempo de depuración.

## 1. La proyección de contenido NO atraviesa `@if`

```html
<cp-app-layout>
  @if (isDesktop()) {
    <app-nav cpAppNav />   <!-- ❌ acaba en el slot POR DEFECTO -->
  }
</cp-app-layout>
```

El bloque `@if` se compila como un contenedor sin atributos, así que Angular lo asigna al slot por defecto y todo lo de dentro se renderiza ahí. El atributo del slot tiene que ir en un **nodo declarado estáticamente**.

**Síntoma:** "no veo el layout" — el aside sale vacío y el contenido aparece en `main`.

**Fix:** quitar el `@if` y gestionar la visibilidad por CSS, o envolver en un elemento estático que lleve el atributo.

### 1b. `<ng-content>` dentro de un `@if` SÍ funciona — lo que rompe es duplicar el selector

**Corregido el 2026-08-31.** Esta nota afirmaba que meter el `<ng-content>` dentro de un `@if` en la plantilla del propio componente rompía la proyección. **No es cierto en Angular 17+**: el contenido se instancia igual y se muestra cuando la rama renderiza.

No confundirlo con el punto 1, que sí es real: ese es del lado del **consumidor** —el *atributo* del slot dentro de un `@if`/`@defer`— y ahí el bloque compila a un contenedor sin atributos y todo cae al slot por defecto.

**Lo que sí rompe, del lado del componente:** que **el mismo selector aparezca en dos `<ng-content>`**, típicamente uno por rama de un `@if/@else`. Solo el primero recibe contenido; el otro sale vacío, sin error.

```html
@if (nuevo()) {
  <ng-content select="[actions]" />   <!-- este recibe -->
} @else {
  <ng-content select="[actions]" />   <!-- ❌ este nunca -->
}
```

**Consecuencia práctica:** si las dos ramas necesitan el mismo contenido proyectado, no se puede ramificar la plantilla. Hay que dejar el `ng-content` una sola vez y mover el resto por CSS. Fue el caso del `wrapper` del backoffice: 93 páginas proyectan en `[actions]` y `[content]`, así que su look legacy se resolvió con knobs y no con `@if`.

Verificado el 2026-08-31 sobre `vertical-slider` y `filter` del backoffice, que llevan años con `ng-content` dentro de `@if` sin duplicar selector, y funcionan.

## 2. Los selectores de atributo no fallan si no importas la directiva

`cpInput`, `cpButton`, `cpNavAction` son selectores de atributo: si olvidas importarlos, Angular **no da error**, simplemente los ignora. (Un selector de elemento como `cp-form-field` sí daría NG8001.)

**Cómo verificar:** buscar el campo `dependencies:[...]` del componente compilado en el bundle, o comprobar que las clases (`.cp-btn`, `.cp-nav-action`) están en el `styles.css` final.

## 3. pnpm 11.x revienta con GitHub Packages

```
pnpm: Invalid time value  at detectMinReleaseAgeViolation
```

GitHub Packages no devuelve el campo `time` que espera el chequeo de `minimumReleaseAge`. Afecta a **frontend-validation**, que lo tiene a `1 day` en `pnpm-workspace.yaml`.

**Funciona:** `pnpm add @corepark/corepark-ui@X.Y.Z --config.minimumReleaseAge=0`
**NO funciona:** añadir la versión a `minimumReleaseAgeExclude` — el crash ocurre antes de consultar la exclusión.

Cualquiera que clone y haga `pnpm install` en frío se topa con esto.

### Actualización 2026-08-31: hay CUATRO versiones de pnpm y se comportan distinto

Publicando `0.0.29` e instalándola en los tres consumidores, el mismo comando dio tres resultados:

| Repo | pnpm | Qué pasó |
|---|---|---|
| design-system | 10.27.0 | — |
| frontend-commerce | 10.27.0 | `pnpm install` normal, sin fricción |
| frontend-validation | 11.6.0 | tiene `minimumReleaseAge: 1 day` explícito → **necesitó `--config.minimumReleaseAge=0`** |
| frontend-backoffice | 11.24.0 | **no** declara `minimumReleaseAge`, pero pnpm 11.24 trae un default y en vez de fallar **añadió solo** `@corepark/corepark-ui@0.0.29` a `minimumReleaseAgeExclude` en `pnpm-workspace.yaml` |

Dos consecuencias:

1. Publicar y usar la versión el mismo día **solo es un problema en validation**, y el flag lo resuelve. La 11.24 ya lo gestiona sola.
2. El `pnpm-workspace.yaml` del BO **acumula una entrada por cada publicación**. Conviene limpiarlo de vez en cuando o poner `minimumReleaseAge: 0` y acabar con el tema.

Unificar las cuatro versiones de pnpm sigue pendiente y explicaría bastante ruido.

## 4. `publish:lib` necesitaba `./`

`npm publish dist/corepark-ui` sin `./` se interpreta como shorthand de repo git y falla con `Permission denied (publickey)`. Corregido a `pnpm publish ./dist/corepark-ui --no-git-checks` (pnpm se niega a publicar con working tree sucio sin ese flag).

## 5. Orden de `styles` en angular.json

`corepark-ui/styles.scss` debe ir **PRIMERO** en el array, para que los overrides de la app ganen sobre el `:root` del DS. Corregido en commerce (estaba al final). Validation ya lo tenía bien.

**Al editar `angular.json`:** hacerlo con Edit quirúrgico, no re-serializando con `JSON.stringify` — el repo usa tabs y un `stringify(a, null, 2)` reformatea las 300 líneas.

## 6. Recompilar y sincronizar tras cambios de API

`pnpm run build && pnpm run sync:<app>` después de **cualquier** cambio de tipos, no solo de CSS. Si no, el consumidor compila contra los `.d.ts` viejos (p. ej. `TS2741: Property 'route' is missing`).

## 7. Backticks en un comentario HTML dentro de un `template:` inline

Los shells `ds-app-layout` de commerce y validation usan `template:` inline (template literal). Un comentario en estilo markdown dentro de él **cierra la cadena**:

```ts
template: `
  <!-- Fed the same `routes` as the rail -->   // ❌ el backtick termina el template
`
```

**El error no dice nada de esto:** `NG1002: Incorrect number of arguments to @Component decorator` + `TS2349: This expression is not callable`, apuntando al `@Component({`. Lo que pasa es que el resto del archivo se parsea como argumentos extra del decorador — la pista real está en el tipo que imprime TS2349, donde aparecen propiedades que no escribiste ahí (`routes: any`) y falta `styles`.

Regla: en templates inline, nada de backticks — ni en comentarios. Texto plano o comillas simples.

## `toSignal` se suscribe DURANTE la construcción — no leer inputs requeridos ahí

```ts
readonly #data = toSignal(
  this.#reload.pipe(
    startWith(null),
    switchMap(() => this.#service.get(this.employee().id))   // 💥 NG0950
  ), { initialValue: null })
```

`toSignal()` se suscribe **mientras se inicializa el campo**, o sea en construcción. Los inputs se asignan **después**, así que un `startWith` que dispara el `switchMap` de inmediato lee un `input.required()` que aún no tiene valor: *"Input X is required but no value is available yet"*.

**Arreglo: que la fuente sea el propio input.** `toObservable` se apoya en un `effect`, que corre ya con los inputs puestos:

```ts
combineLatest([toObservable(this.employee), this.#reload.pipe(startWith(null))]).pipe(
  switchMap(([employee]) => this.#service.get(employee.id))
)
```

**Cómo detectarlo leyendo:** en un archivo con `toSignal` **y** `input.required`, mirar si la tubería del `toSignal` lee el input. Si la fuente es un observable de estado o de servicio, no hay problema — solo falla cuando la fuente emite síncronamente y toca un input.

Cazado el 2026-09-01 en `pin-codes-panel` (el panel expandible de `/employee-center`); reventaba en cada fila abierta. En el BO no hay ningún otro caso: los otros tres archivos con las dos cosas parten de observables de estado.

## Los estilos encapsulados NO alcanzan el contenido proyectado

Un nodo proyectado conserva el atributo de encapsulación **del consumidor que lo declara**, no el del componente que lo recibe. Así que `cp-table` no podía colocar lo que le proyectaban en su toolbar: ninguna regla suya casaba con ese nodo.

**Salida: que el componente envuelva su propio slot** en un elemento que sí posee, y estilar ese.

```html
<div class="cp-table-toolbar__aside">   <!-- este sí lleva el atributo de la lib -->
  <ng-content select="[cpTableToolbar]" />
</div>
```

La alternativa —que cada consumidor repita la regla de colocación— es exactamente la deriva que un slot existe para evitar.

## Una entidad HTML dentro de una cadena de TS no se decodifica

```html
{{ cond ? '&#9679;' : '&#9632;' }}   <!-- 💥 imprime el texto literal -->

@if (cond) { &#9679; } @else { &#9632; }   <!-- ✅ markup, sí se decodifica -->
```

Las entidades solo se decodifican escritas como markup en la plantilla. Dentro de un literal de TypeScript son texto y nada más. **El build no dice nada**; se ve en pantalla.

Ver [[feedback_package_manager]], [[project_migration_status]] y [[project_bo_module_migration]].
