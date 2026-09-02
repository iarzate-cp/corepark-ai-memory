---
name: pending-work-estado-al-cierre-del-2026-08-31
description: qué queda abierto por repo; sustituye el listado del 28 de agosto
metadata: 
  node_type: memory
  type: project
  originSessionId: 02d8be60-b4d7-4d73-99ae-4bda6564aa7f
  modified: 2026-09-01T23:04:32.120Z
---

Reescrito el **2026-08-31**. El histórico de lo hecho vive en [[project_responsive_shell]], [[project_migration_status]], [[project_route_titles]] y [[project_backoffice_wrapper]]; esto es solo **lo que queda abierto**.

## Transversal

**Estado al cierre del 2026-08-31.** Todo commiteado y empujado; los cuatro working trees limpios.

| Repo | Rama | Versión |
|---|---|---|
| design-system | `develop` | **`0.0.30` publicada** (2026-09-01) |
| frontend-commerce | `feature/design-toggle` + `feature/staging` | `2026.8.31`, lib `0.0.29` |
| frontend-backoffice | `feature/design-toggle` + `feature/staging` | **`2026.9.1`, lib `0.0.30`** |
| frontend-validation | `feature/auth-layout-ds` + `feature/design-toggle` → `develop` | `2026.8.0` ⚠, lib `0.0.29` |

**`0.0.30`** lleva el trabajo de tabla de la migración de módulos del BO: `rowEmphasis`, `singleExpand`, el slot `[cpTableToolbar]` con `searchPlaceholder`, los dos arreglos de layout del toolbar, y los contornos de tarjeta a `text-200`. **Commerce y validation siguen en `0.0.29`** — no se han actualizado, y no lo necesitan salvo que quieran esas APIs.

**Ninguna rama está mergeada a `main` en ningún repo.** GitHub ofrece abrir PR en todas.

- **Los `.npmrc` de los tres consumidores llevan token en texto plano y trackeado en git.** Israel lo ha escalado repetidamente para meterlo en AWS y le han ignorado — **no es responsabilidad nuestra, no volver a sacarlo.**
- **Nada del trabajo de diseño está verificado en pantalla más allá de lo que Israel comparó a mano.** El BO tiene su lista concreta en [[project_design_toggle_backoffice]]; commerce entero está sin ver.
- `pnpm test` **no arranca en commerce** (falta `jest-environment-jsdom`, tampoco en `main`) y el **BO no tiene script de test**. La única suite que corre es la de la librería: 308 tests.

## design-system — rama `develop`

### Bloqueante para el siguiente paso de responsive

- **Migrar `cp-menu` a overlay del CDK.** Hoy pinta su panel inline con `position: fixed`, lo que (a) lo deja atrapado en el contexto de apilamiento de su ancestro y (b) **bloquea meter container queries en el área de contenido**, que es lo correcto para `cp-table`, `stat-card` y los módulos. Ver [[project_responsive_shell]].

### Deuda de la lib

- **Contradicción tipográfica**: `CLAUDE.md` especifica page title 20px/700 y `cp-page-header` quedó en 24px/600. Decidir cuál manda ([[project_cp_page_header]]).
- Gap del `cp-menu-item`: su `disabled` hace `stopPropagation()`, que **no** bloquea el `(click)` del consumidor. Hoy se blinda con un early return en cada consumidor; arreglarlo en el DS.
- Eje **`GRAD`** de Roboto Flex sin habilitar. Son tres `index.html`.
- Subtítulo de `cp-auth-layout` bajo WCAG AA en claro (~3.5:1).
- El padding del subscript de `cp-form-field` quedó en `0.5rem`; el inset del campo es `0.75rem`.
- **Rail icon-only para la banda 768–1024**: descartado a propósito (haría falta submenús flyout), es el siguiente paso natural si 32rem de contenido a 768px resulta estrecho.

### Cobertura de tests — `pnpm test:coverage` FALLA

Umbral del **80%** en `vite.config.mts`; la lib está en **57.3% stmts / 47.5% branches / 52.4% funcs / 58.5% lines**. `pnpm test` pasa porque no mide.

- **Dos exclusiones obsoletas en `vite.config.mts`**: `**/layout/**` es una **exclusión muerta** (esa ruta se movió a `layouts/shell-layout/`), y `**/tooltip/**` ahora se come también `directives/tooltip/`, que no era la intención.
- **Los peores huecos están excluidos, no en 0%** — así que no aparecen en el informe: `dialog/**` (5 archivos, incluido `dialog-service`, la API imperativa más usada), `tooltip/**` (3), `select/**` (2), `date-range-picker/**` (2), `sidebar/**` (2), `notification-service` + `host`, charts bar/donut/line, `table/cell-def` + `expand-def`.
- **A 0% con lógica real**, por valor: `time-picker` (66 líneas, la unidad más grande sin test), `menu-trigger-directive` + `menu-component` (46, y ahí vive el bug del `disabled`), `color-scheme-service` (21, de él depende el theming de las tres apps), `stacked-bar-chart` + `trend-chart` (47, **no están excluidos**, cuentan contra el umbral), los tres layouts, `modules/**` (9 componentes, ~160 líneas).
- Orden de retorno recomendado: `color-scheme-service` → `menu` (y arreglar el bug del `disabled`) → `dialog-service` → `time-picker`. Los `modules/**` conviene **excluirlos explícitamente** en vez de fingir que se cubrirán.
- Ya cubiertos y no hace falta volver: `cp-app-nav` (9 specs, nuevos), `cp-app-nav-bar` (12), `cp-page-header` (8).

## frontend-backoffice — ramas `feature/design-toggle` y `feature/staging`

El toggle de diseño y sus seis fallos visuales están en [[project_design_toggle_backoffice]]. La reconstrucción de módulos sobre la lib, en [[project_bo_module_migration]].

### Migración de módulos — seis hechos, el resto sin empezar

Hechos: dashboard, `analytics/activity-by-rate-class`, `locations`, `locations/partners` (absorbe request points), `employee-center` (absorbe pin codes), `guest-settings/profiling`.

**Sin reconstruir** — cada uno sigue con su entrada de ruta única, así que no urge nada: `analytics/trend`, `settings/*` (rates, stations, …), `reports/*`, `payments/*`, y las 46 hijas restantes de la raíz.

**Tarea que Israel dejó para el final:** refactor de `daily-report-detail` (780 líneas, compartidas entre `/locations` y `daily-detail-dialog`). **Solo código, sin rediseño y sin módulo nuevo** — lo comparten los dos diseños, así que ni el comportamiento ni el markup visible cambian.

**Deuda que deja la migración:**
- Las tablas de pin codes y de coches **clásicas** (`pin-codes-table`, `guest-profile-cars`) quedan sin usar bajo el diseño nuevo, pero siguen sirviendo al clásico — no se borran.
- **La contraseña del partner se muestra en claro y permanente** en la ficha. El clásico la enseñaba también, pero tras un clic. Enmascararla con revelar es decisión de producto: no se sabe si soporte la lee en voz alta.
- Claves i18n nuevas solo en `en`/`es`: `REQUEST_POINTS.OF_PARTNER`, `PARTNERS.DETAIL.EDIT_BUTTON`, `PARTNERS.DETAIL.LOCALITY`, `PIN_CODES.LOADING`. `id.json` no tiene bloque `REQUEST_POINTS`.

- **Sin usar y no borrados** (política establecida): `app-nav-mobile`, `app-nav` local, `app-nav-icon`, `hamburguer`, `main-layout`, `corepark-imagotype`, `auth-layout-bg.webp`, y los cuatro componentes de icono que usaba `user-settings` (`video-guides-icon`, `lang-icon`, `sign-out-icon`, `user-icon`).
- **Traducciones por confirmar**: `NAVIGATION.MENU` y `NAVIGATION.ARIA_LABEL` en es/id las escribí yo (chrome, no copy de producto). `id.json` sigue casi sin bloque `NAVIGATION`, así que el nav en indonesio muestra claves crudas.
- **`_router.scss`**: un `.loader-container` a z-index 1000 con `backdrop-filter`, que además de tapar el rail **crea bloque contenedor para `position: fixed`**.
- **`styleComponent` del wrapper es inerte** (4 call sites).
- **Material: 495 archivos.** El rail, la navegación móvil y auth están limpios; el lote natural es `form-field` (97) + `input` (91).
- **NgModules: 61 archivos**, 50 componentes con `standalone: false`.
- **NO tocar la familia de diálogos** (`wrapper-dialog` 259 usos, etc.) salvo de uno en uno.

## frontend-commerce — ramas `feature/design-toggle` y `feature/staging`

El toggle de diseño está en [[project_design_toggle_commerce]].

- **`pms/connections` y `pms/arrivals-file` no migradas**: conservan su header a mano. Tienen `ownHeader: true`, así que no hay header doble.
- `searchable-dropdown` (z-index 300) y `date-range-field` (400) son contenido de página **por encima** de la barra inferior en reposo (200): un panel abierto pintará sobre ella. Puede ser correcto; mirar en móvil.
- **`page-label.component.ts`** es una tercera copia del árbol de rutas y solo lo usa el `main-layout` muerto. `MainLayoutState` y su `@HostListener('window:scroll')` también son de ahí.
- Usa su propio `ColorSchemeState` en vez del `ColorSchemeService` del DS.

## frontend-validation — ramas `feature/auth-layout-ds` y `feature/design-toggle`

**`feature/design-toggle`** (2026-09-01) nació de `main` y mergeó `feature/auth-layout-ds`. **Empujada, y `develop` avanzó a su punta por fast-forward** hasta `3e24a86`: el toggle y el diseño nuevo ya no están aislados en una rama. Validation **no tiene `feature/staging`**; su rama de integración es `develop`. Ver [[project_design_toggle_validation]].

- **Se quedó atrás en versionado**: sigue con el script del **contador** y en `2026.8.0`, o sea con una cadena con forma de fecha que no es la fecha. Traerle el script por día de commerce/BO es un `cp` — ver [[project_calver_apps]].
- ~~**No tiene toggle de diseño.**~~ Lo tiene desde el 2026-09-01, pero **solo en `feature/design-toggle`**; `feature/auth-layout-ds` sigue migrada a la fuerza sin convivencia.

- `CdkLayoutState.maxWidth` **mide el viewport, no el área de contenido**, así que en la banda 768–1030 va desajustado ahora que el rail ocupa ~248px ahí. Es el caso legítimo de container query — bloqueado por `cp-menu`.
- `isBrowser` (1030px) de `CdkLayoutState` no lo usa ningún componente vivo, solo el `main-layout` muerto. El nombre además miente: mide ancho, no entorno.
- El `<h6>Enter your ticket number</h6>` de `home` **duplica el subtítulo** de la ruta. Copy visible, decisión de Israel.
- `request-car` y `overnight-report` no tenían cabecera: el header ahí es UI nueva, conviene mirarlo con ojo de diseño.
- Los layouts viejos `main-layout` y `auth-layout` **no se borran** (decisión de Israel). Quedan los 3 fondos negros hardcodeados de `main-layout/ui/`.

## frontend-valet-web

**Fuera de todo esto por decisión de Israel. No proponer sincronizarlo.**

## Versionado

CalVer en las tres apps ([[project_calver_apps]]). Pendiente: **estampar el SHA corto** junto a la versión, y automatizar el bump.
