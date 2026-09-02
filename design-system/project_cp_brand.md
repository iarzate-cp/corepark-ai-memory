---
name: cp-brand-el-lockup-de-marca-del-rail
description: componente del DS que unifica wordmark + isotipo + nombre de proyecto; sustituye los bloques .brand a mano de las tres apps
metadata: 
  node_type: memory
  type: project
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-27T23:44:20.098Z
---

Creado el **2026-08-27**. `lib/components/brand/`.

Antes el lockup estaba en tres sitios distintos: commerce y validation cada una con un bloque `.brand` a mano (mismo wordmark + isotipo, mismas reglas de espaciado duplicadas), el backoffice con un `corepark-imagotype` local, y el nombre del proyecto en el input `product` de `cp-app-layout`.

## API

```html
<cp-brand project="CRM" />
```

| Input | Default | |
|---|---|---|
| `project` | `''` | Nombre del producto bajo el lockup. Oculto si vacío |
| `name` | `'core'` | Mitad en peso light |
| `accent` | `'park'` | Mitad en bold, color primary |
| `isotypeHeight` | `'2.25rem'` | |
| `isotypeColor` | `var(--cp-ui-color-text-950)` | Por defecto sigue al wordmark |

Knobs: `--cp-ui-brand-{gap,word-size,color,accent-color,project-color}`.

**Presentacional, no lleva enlace.** El consumidor lo envuelve donde el rail deba navegar a home: `<a cpAppBrand routerLink="/"><cp-brand project="…" /></a>`, y el `<a>` solo aporta `display:inline-block; color:inherit; text-decoration:none`.

**Sin gap en el lockup, y el template cierra las etiquetas pegadas (`><`)** para que no entren nodos de texto: "core" y "park" tienen que leerse como una palabra y el isotipo va a ras. Un `gap` de flex separaría los tres por igual y partiría el wordmark.

## Marca blanca: NO hay slot

Consideré un `[cpBrandMark]` proyectable y **lo quité antes de usarlo**: la proyección no atraviesa `@if`, así que el wrapper tendría que estar siempre presente, y entonces un `:has([cpBrandMark])` ocultaría el lockup de corepark incluso sin marca blanca. Y al revisarlo, **nadie lo necesitaba**: el backoffice ya ramifica con `@if/@else` antes de llegar al componente.

El patrón para marca blanca es ese: ramificar fuera, y etiquetar el rail con el `product` de `cp-app-layout` para que el nombre sobreviva las dos ramas.

## Estado en las tres apps

- **backoffice** — `operator-logo` conserva su `@switch` de 12 logos de operador; su rama `@else` usa `<cp-brand />` sin `project`, porque el "BACKOFFICE" sigue viniendo del `product` del layout (así aparece también sobre el logo del operador). `corepark-imagotype` queda **sin usar, no borrado**.
- **commerce** — `<cp-brand project="CRM" />`, fuera el `product="CRM"` del layout.
- **validation** — `<cp-brand project="Partner" />`, fuera el `product="Partner"`.

**El `product` de `cp-app-layout` sigue existiendo** y lo usa el backoffice. Para apps cuya marca es siempre corepark, `project` de `cp-brand` lo supersede — el nombre pertenece al lockup, no al chasis.

## Ojo al migrar el backoffice: el lockup se hizo más pequeño

`cp-brand` usa `--cp-ui-font-size-subtitle-1` (1.875rem) y el isotipo a `2.25rem`. El `corepark-imagotype` local usaba `--fs-extra-big` (2rem) y el isotipo a su default de 3rem. Un 14% de los píxeles del bloque de marca cambian — es deliberado, son los valores que commerce y validation ya pasaban a mano.

## Compensación óptica de peso en dark

Israel reportó que en validation "core" se veía **ligeramente más gruesa** que en el backoffice y commerce. Se descartó todo lo mecánico midiendo: mismo `font-weight: var(--cp-ui-font-weight-light)` en el CSS compilado de los tres, el token resuelve a **300** en los tres con una sola definición y sin overrides, y suavizado (`-webkit-font-smoothing: antialiased`) y carga de Roboto Flex idénticos.

La causa es perceptual: **validation corre en `data-theme="dark"`**, así que "core" es `#ffffff` sobre un rail `#272727`, mientras el backoffice lo pinta `#1E2126` sobre `#FAFAFC`. Blanco sobre oscuro se lee más grueso al mismo peso — halación (las zonas claras sangran sobre las oscuras) más la asimetría del antialiasing de macOS. Solo se nota en "core" porque "park" va en bold y tintado.

Arreglo en el DS: knob `--cp-ui-brand-word-weight` (default `--cp-ui-font-weight-light`) y

```scss
:host-context([data-theme='dark']) { --cp-ui-brand-word-weight: 250; }
```

**250 es un punto de partida, no un valor medido** — el ajuste fino es a ojo en pantalla. El eje `wght` de Roboto Flex es continuo, así que se puede afinar sin saltar un paso; con una fuente estática esto no se podría hacer.

**Nota de verificación:** `:host-context` viaja **sin resolver** en el paquete (compilación parcial de ng-packagr) y lo transforma el compilador de cada app. Comprobado en el build de validation: `[data-theme=dark][_nghost-%COMP%], [data-theme=dark] [_nghost-%COMP%]`.

Ver [[project_cp_app_layout]] y [[project_migration_status]].
