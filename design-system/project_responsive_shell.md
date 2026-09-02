---
name: project_responsive_shell
description: "responsive del shell — estado actual: rail a 768, cp-app-nav-bar de dos triggers, modelo de scroll, capas z, y por qué las container queries están bloqueadas"
metadata: 
  node_type: memory
  type: project
  originSessionId: 02d8be60-b4d7-4d73-99ae-4bda6564aa7f
  modified: 2026-08-31T20:34:41.478Z
---

Trabajado el **2026-08-31** en `~/Dev/design-system` (rama `develop`) y aplicado a los tres consumidores. Esto es **estado actual**, no diario; la historia que aún enseña algo está al final.

## El bug de fondo que se arregló

`cp-app-layout` tenía **un solo breakpoint y era binario**: a `$laptop` (1024) el rail hacía `display: none` y debajo no había navegación salvo que el consumidor llenara `[cpAppMobile]`. **Solo el backoffice lo llenaba** → commerce y validation se quedaban literalmente sin navegación bajo 1024px.

## Estado actual del shell

| Banda | Navegación |
|---|---|
| ≥ `$tablet` (768) | rail completo, 15.5rem |
| < `$tablet` | rail oculto, `cp-app-nav-bar` |

**No hay rail icon-only en tablet, y es deliberado.** El rail canónico (Material 3) asume 3–7 destinos planos; este nav son grupos con hijos, así que icon-only exigiría submenús flyout para no dejar a los hijos inalcanzables. Es un componente nuevo, no un pulido. Sigue siendo el paso natural si 32rem de contenido a 768px resulta estrecho.

### Modelo de scroll

**El documento no scrollea; scrollean las regiones.** `--cp-ui-app-layout-height: 100svh` + `overflow: hidden` en el host; el `main` scrollea con `--cp-ui-app-layout-main-overflow: auto`. **Los dos knobs se mueven juntos** — poner solo `height: auto` recorta.

Tres consecuencias:

1. **`position: sticky` en una página** se pega al scrollport del `main`, no al borde del documento. Mejor, pero es un cambio de comportamiento.
2. **`cdkScrollable` en el `<main>` es obligatorio.** Los overlays con `reposition()` (tooltip, select, date-picker del DS, y todo overlay de Material del consumidor) escuchan por el `ScrollDispatcher` del CDK; un scroller sin registrar deja el panel flotando sobre contenido que ya se movió. Es el primer `imports: []` que tiene `cp-app-layout`.
3. Comprobado que en el BO **no hay ni un listener de scroll de `window`**, así que nada dependía del scroll del documento.

**`svh`, nunca `dvh`, para lo fijo:** con `dvh` el shell crece y encoge con el chrome del navegador móvil, relayouteando la página a mitad de scroll.

### El fade del rail sin dejar el primer item apagado

**El truco: la distancia del fade ES el padding vertical del área de scroll.**

```scss
--cp-ui-app-layout-nav-fade: 1.25rem;
--cp-ui-app-layout-nav-padding: var(--cp-ui-app-layout-nav-fade) var(--cp-ui-spacing-4);
mask-image: linear-gradient(to bottom, transparent 0, #000 var(--fade),
                            #000 calc(100% - var(--fade)), transparent 100%);
```

En reposo el gradiente cae sobre ese inset vacío, así que **nada se atenúa**; al scrollear el padding se va y el contenido entra en la banda. Sin listener y sin `animation-timeline: scroll()` (soporte insuficiente en Safari). El knob a `0` colapsa el mask a opaco.

**Había dos scrollers anidados** y se colapsó a uno: `.cp-app-layout__nav` y el `:host` de `cp-app-nav` tenían ambos `overflow-y: auto`. El del componente se quitó — el contenedor es quien scrollea, y es el único que puede enmascarar sus bordes.

## `cp-app-nav-bar` — API y forma final

**Dos triggers, siempre los mismos dos, en las tres apps.** Decisión de Israel.

- **Menú** → hoja con el árbol completo (`cp-app-nav [expanded]="true"`).
- **Cuenta** → hoja con lo proyectado en `[cpNavBarAccount]`, el mismo `user-settings` que el rail.

Inputs: `routes` (required), `menuLabel`, `accountLabel`, `closeLabel`, `navLabel`. **No hay `max` ni `routeClick`.** Nada refleja la ruta actual: con dos triggers genéricos no hay destino que marcar, y el nav de la hoja ya marca el activo. Tocar el trigger de la hoja abierta la cierra; el otro conmuta en un toque.

Vive en `components/app-nav/` junto a `app-nav-footer` — misma feature, sin carpeta nueva.

### Invariante que NO se puede romper

**Las dos superficies viven siempre en el DOM y se muestran por clase `.is-open`.** Hay un solo `<ng-content>` para el bloque de cuenta, así que —**corregido el 2026-08-31**— un `@if` habría funcionado: la razón que se dio aquí era falsa (ver [[feedback_angular_gotchas]] 1b). La implementación se mantiene por otro motivo, que sí vale: es determinista y no depende del motor de animaciones para que la hoja se desmonte. Por eso también se borró la animación de Angular (`SLIDE_UP_SHEET`) y las transiciones son CSS sobre `transform` + `visibility`.

`visibility` carga el estado cerrado: oculta, **saca el subárbol del orden de tabulación** y es animable (a diferencia de `display: none`). El bloque de cuenta sí usa `display: none` — con `visibility` dejaría una caja vacía sobre el menú.

Hay un spec que fija esto (*"keeps the projected account block mounted while the menu is showing"*). **No borrarlo.**

## Capas z — tres defectos distintos, no confundirlos

1. **El rail estaba en `--cp-ui-z-raised` (10)**, el suelo de la escala. El BO tiene contenido de página en 40, 50, 90, 100 (×7, tres en `_tabs.scss`), 110, 130, 200 y 300 → todo eso lo tapaba. Ahora **`--cp-ui-z-sticky` (200)**, el token documentado como *"sticky sidebars, headers"*. Mismo valor que `cp-app-nav-bar`; nunca coexisten.
2. **El backdrop de la hoja tapaba la propia barra**: barra en 200, backdrop en `z-overlay` (300) con `inset: 0`. Arreglo: `[class.is-sheet-open]` sube la barra a `z-modal` **solo mientras la hoja está abierta** — en reposo es chrome y un modal de verdad debe taparla.
3. **La cabecera sticky de una hoja necesita `z-index`**, no le basta el fondo. El nav que va después está lleno de posicionados (`.cp-app-nav__children` es `relative`, su marcador `absolute`), caen en el mismo paso de pintado con `z-index: auto`, y **decide el orden del árbol: gana el que va después**. Pasaba en `app-nav-mobile` del BO (texto cruzando la X) y estaba latente en `cp-app-nav-bar__sheet-bar`.

**Y el detalle que hace que subir números no sirva:** `cp-menu` pinta su panel **inline, no portalado a `body`**. El `<aside>` es `position: fixed` **con** z-index → contexto de apilamiento, así que todo lo de dentro queda atrapado en ese nivel pida lo que pida. Sus `998/999` mágicos se sustituyeron por `--cp-ui-z-dropdown` con knob `--cp-ui-menu-z-index`.

Siguen por encima del rail en el BO, y correctamente: `_popover` (300), `_tooltips`, `_loader`, `_slider-dialog`. **Ojo con `assets/scss/components/_router.scss`**: un `.loader-container` a 1000 con `backdrop-filter`, que además de tapar **crea bloque contenedor para `position: fixed`**.

## BLOQUEADO: container queries en el área de contenido

Lo correcto sería `container-type: inline-size` en `.cp-app-layout__main` para que `cp-table`, `stat-card` y los módulos midan su caja real en vez del viewport. **No se debe hacer todavía:** `container-type` implica `contain: layout`, que **crea bloque contenedor para descendientes `position: fixed`**, y el panel de `cp-menu` es `fixed`. Rompería en silencio todo `fixed` dentro del contenido de página.

**Precondición: migrar `cp-menu` a overlay del CDK.** Hasta entonces, media queries.

Otro límite: **`var()` no resuelve dentro de `@container`**, igual que en `@media`.

## Convergencia de las tres apps

**Las tres tienen ahora un `user-settings`** que envuelve `cp-app-nav-footer` con `:host { display: contents }` y se renderiza **dos veces**: `cpAppUser` del rail y `cpNavBarAccount` de la hoja. Antes el footer estaba inline en el shell de cada app, así que **por debajo de 768 no había forma de llegar al tema, al sign out, al chat de validation ni al OTP de commerce**.

- **BO**: fuera `app-nav-mobile`. Con él mueren `app-nav` local, `app-nav-icon` y `hamburguer` — **sin usar, no borrados** (política establecida). Desapareció también el trigger de CorePark, que **no hacía nada**: su `@if` no tenía rama, abría una hoja vacía. Cero Material en toda la navegación.
- **Etiquetas del BO**: `NAVIGATION.MENU`/`CLOSE`/`ARIA_LABEL` nuevas en en/es/id, y `USER_SETTINGS.TITLE` añadida a `id.json` (`Pengaturan`, copiado de otra clave ya traducida). "Close" también se copió de traducciones existentes. **"Menu" y "Navigation" en es/id las traduje yo** — chrome, no copy de producto, pero conviene confirmarlas. `id.json` sigue casi sin bloque `NAVIGATION`, así que el nav en indonesio muestra claves crudas de antes.
- **BO breakpoints**: sus tres `1024` alineados al `$laptop` del DS pasaron a `768`, y se corrigieron dos off-by-one (`isDesktop` era `> 1024` y el nav móvil se ocultaba con `> 1024px`, mientras el CSS muestra el rail **desde** 1024). `AppLayoutState` pasó de rastrear `innerWidth` con `debounceTime` a `matchMedia` — ver [[feedback_signals_over_observables]].
- **commerce**: `MainLayoutState` y su `@HostListener('window:scroll')` solo los usa el `main-layout` muerto. Limpio.
- **validation**: `CdkLayoutState` sí lo usan dos componentes vivos, pero solo `isSmall` (550px) y `maxWidth`. Este último **mide el viewport, no el área de contenido**, así que en 768–1030 va algo desajustado — es el caso legítimo de container query, bloqueado.

## Verificación: qué está probado y qué no

- **308 tests, 24 archivos.** `cp-app-nav-bar` y `cp-app-nav` tienen specs propios (este último no tenía ninguno, estaba al 45.7% de rebote).
- En el `dist` se verificó que sobreviven `env(safe-area-inset-bottom,0px)` (minificado sin espacio, fallback intacto), `100svh`, el `mask-image` con sus `calc()`, `cdkScrollable` y `max-width:47.9375rem`.
- Los tres consumidores: build verde y `diff -rq` vacío contra el `dist`.
- **Nada verificado en pantalla.** Todo el trabajo de la hoja móvil vive en el modo `expanded`, que solo existe ahí.

## Lo que la historia enseña

**El bug de "todo activo" en la hoja.** `[class.is-active]="isOpen(route)"` e `isOpen` devuelve `true` para todos los grupos cuando `expanded`. En el rail colapsable "abierto" y "actual" **coinciden** (abrir navega al primer hijo), así que el binding era correcto **hasta que existió un segundo modo**. Arreglado con `isGroupActive`, que en `expanded` usa `holdsUrl`. `aria-expanded` sigue con `isOpen`, que ahí sí corresponde.

Es el patrón de fallo más difícil de ver leyendo un diff: **código correcto que deja de serlo al añadir un modo.**

**Dos diseños de la barra que Israel rechazó**, y por qué:

1. Tabs = primeros N del array, hoja = solo el overflow → **el menú quedaba partido en dos superficies** y había que recordar en qué lado estaba cada cosa.
2. Hoja con todo, tabs solo de rutas planas → mejor, pero **una barra distinta por app** (commerce 2, validation 4, BO 3) y el footer enterrado dentro del menú.

**La lección:** derivar la forma de un componente de los datos del consumidor suena elegante y produce inconsistencia entre productos. La barra es chrome, y el chrome debe ser idéntico en toda la suite. Cuando la pregunta es *"¿por qué esto se ve distinto entre apps?"*, la respuesta rara vez es *"porque sus datos difieren"*.

**Método que vale repetir:** tras escribir un test de regresión, **reintroducir el bug y comprobar que falla**. Se hizo con el de "todo activo" (fallan 2 con `['Settings','Reports']` en vez de `[]`). Un test que pasa antes y después no prueba nada.

Ver [[project_cp_app_layout]], [[project_layouts_category]], [[project_pending_work]], [[feedback_angular_gotchas]].
