---
name: project_cp_page_header
description: "cp-page-header — API, slots, el límite de proyección del header en el shell, y las decisiones tipográficas"
metadata: 
  node_type: memory
  type: project
  originSessionId: aea67aa5-d32a-4d47-b014-93e055e5f1b9
  modified: 2026-09-01T23:05:41.256Z
---

Creado el **2026-08-28**. `lib/components/page-header/`. Publicado en las tres apps vía sync.

Extraído del `wrapper` del backoffice **invirtiendo el control**: aquel resolvía su título de un mapa central `URL → clave i18n` y elegía qué iba bajo el título con dos arrays de rutas cableados. Aquí lo suministra el consumidor.

## API

```html
<cp-page-header heading="Ticket transactions" subheading="Last 30 days">
  <cp-breadcrumb cpPageMeta />
  <button cpButton cpPageActions type="button">Export</button>
</cp-page-header>
```

- **Inputs**: `heading`, `subheading`. Ocultos si vacíos. El heading llega **ya traducido** — la lib no toca i18n.
- **Slots**: `[cpPageMeta]` (bajo el título), `[cpPageActions]` (derecha). **No hay slot por defecto**: el contenido de la página va *después* del elemento.
- **Cero padding**, solo `margin-block-end`. El inset lo pone `cp-app-layout` (ver [[project_cp_app_layout]]).
- Apila por debajo de **768px** y ahí las acciones van a `width: 100%`.

Los slots son atributos planos, así que **un typo se traga el contenido sin error**.

## Tipografía — cambiada el mismo día

Nació con el look del BO (24px / light 300). Israel pidió más carga visual:

| | Inicial | Ahora |
|---|---|---|
| Peso | `light` 300 | **`semibold` 600** |
| Tracking | 0 | **`tight` -0.01em** |
| Subtítulo | `paragraph-sm` 12px | **`paragraph-md` 14px** |

**El peso no es la única palanca**: cerrar el tracking hace que la palabra se lea como masa densa. El subtítulo subió por equilibrio — a 12px bajo un título de 24/600 se leía como pie de foto.

**No se compensó el modo oscuro** (a diferencia de `cp-brand`, ver [[project_cp_brand]]): la halación solo se nota en pesos ligeros.

**Deuda abierta**: `CLAUDE.md` especifica page title **20px/700** y el componente quedó en **24px/600**. Uno de los dos hay que corregir.

**`GRAD` sería la herramienta correcta** para engordar el trazo sin mover métricas (no cambia los anchos de avance), pero las tres apps piden solo `opsz,wght` a Google Fonts. Habilitarlo son tres `index.html`.

## El límite que descubrió commerce

**Un header pintado por el shell no puede recibir acciones proyectadas desde la página ruteada.** La proyección va de padre a hijo, y ahí la página es hermana del header.

Por eso commerce tiene `data: { ownHeader: true }`: la página monta su propio `cp-page-header` leyendo `heading()`/`subheading()` de la ruta, y el shell no pinta nada. Ver [[project_route_titles]].

**El BO adoptó el mismo flujo el 2026-09-01, con el default invertido** mientras dure su migración: `ownHeader` arranca en `true` porque 140 plantillas siguen usando `wrapper`, y un módulo reconstruido se apunta con `ownHeader: false`. Ver [[project_bo_module_migration]].

**Es dueño del espacio que hay debajo** (`margin-block-end: var(--cp-ui-page-header-gap)`, spacing-8). La página **no** debe añadir otro — un `gap` en su host lo suma y el header queda al doble de distancia que en el resto. Si hace falta otra medida, el sitio es el knob.

## Verificación

8 specs en `page-header-component.spec.ts`, incluidos **los dos que importan**: que el contenido aterriza en el slot marcado, y el caso de **doble proyección** (página marca `actions` → shim re-proyecta como `cpPageActions`), que es la forma que usan las 93 páginas del BO. Un build verde no prueba nada de esto.
