---
name: project_material_to_corepark_ui
description: "Sacar Angular Material del backoffice y sustituirlo por corepark-ui: censo por paquete, qué hay hecho, y el orden por dependencia"
metadata:
  node_type: memory
  type: project
  modified: 2026-09-03
---

Encuadre que dio Israel el **2026-09-03**: «el ejercicio ahora es básicamente quitar angular material de todos los módulos y poner los de `@corepark/corepark-ui`».

**Aplica la regla de [[feedback_redesign_solo_new_design]]**: cada módulo se sustituye en un `ds-*` nuevo con ruta duplicada, nunca en sitio. El clásico no se toca.

## El censo (medido el 2026-09-03)

**504 ficheros** de `src/app` importan `@angular/material`. Por paquete, en ficheros:

| Material | ficheros | Equivalente en la lib |
|---|---|---|
| `dialog` | 334 | `DialogService` + `cp-dialog-content` + `DialogRef` + `DIALOG_DATA` |
| `snack-bar` | 224 | `NotificationService`, vía la fachada `NotifierService` |
| `form-field` + `input` | 100 + 94 | `cp-form-field` + `cpInput` |
| `icon` | 68 | `@tabler/icons-angular` (ya instalado) |
| `table` + `paginator` + `sort` | 62 + 52 + 26 | `cp-table` — los tres en uno |
| `select` + `autocomplete` + `chips` | 38 + 21 + 2 | `cp-select` — los tres |
| `menu` | 35 | `cp-menu` |
| `checkbox` + `slide-toggle` + `radio` | 35 + 16 + 5 | `cp-checkbox`, `cp-switch`, `cp-radio-group` |
| `tooltip` | 16 | `cpTooltip` |
| `datepicker` | 16 | `cp-date-range-field` / falta fecha única |
| `core` (MatRipple) | 12 | `cpRipple` |
| `button` | 40 | `cpButton` |

**Muertos, solo en `material.module.ts`:** `slider`, `expansion`, `card`, `list`. Se van al recortar el barril.

**Huecos con uso real:** picker de **fecha única** (15 ficheros), spinner (3), stepper (1), button-toggle (1). Y `mat-autocomplete` **asíncrono** — `google-searcher` busca contra Google Places y `cp-select` filtra un array estático, así que ése no encaja.

## Hallazgos que cambian el plan

**`cpInput` es una directiva vacía** (`host: { class: 'cp-field-input' }` y nada más). Así que `matInput` → `cpInput` funciona con reactive forms sin tocar nada: el bloque más grande es el más fácil.

**`cp-select` es un combobox con buscador**, con CVA y overlay del CDK. Se come select, autocomplete y chips de una vez.

**El CSS estructural del overlay del CDK lo traía el tema de Material.** Sin él, `cp-select`, `cp-menu` y las notificaciones se renderizan como un bloque estático al final del `<body>` — en silencio. Arreglado en la lib (`613cf28`): su `styles.scss` ya carga `@angular/cdk/overlay-prebuilt.css`. **Era el bloqueante que habría roto todo al borrar `custom-theme.scss`.**

## Deuda propia que el ejercicio desentierra

**Cuatro sistemas de botón**: `actioner` (371 ficheros, `_actioner.scss` 313 líneas), `ui-button` (41, 121 líneas), `cpButton` (14), `mat-button` (6). Más `_cp-button.scss`, 74 líneas de SCSS local imitando al DS. **508 líneas de CSS de botón propio; Material es 6 ficheros de esos.**

**Tres sistemas de icono**: `mat-icon` (69), `tabler-icon` (23) y **21 componentes de icono a mano** en `shared/icons/`.

**`nxt-pick-datetime` (19 ficheros)** es un tercer sistema de fechas, ni Material ni DS.

**`@angular/cdk` se queda**: la lib lo necesita.

## Y lo que no sale contando `mat-*`

- **`custom-theme.scss`** (42 líneas) usa `mat.all-component-themes` con la API **M2 legacy**: emite **1.416 custom properties `--mat-*`** y 274 selectores `.mat-*` en un `styles.css` de 149 KB. Se borra al final, cuando no quede ningún componente Material.
- **14 ficheros SCSS pisan `.mat-*` / `mat-mdc-*`**. Mueren con el tema.
- **`MaterialModule`** (34 módulos re-exportados, 24 consumidores) + 4 `*-material.module.ts` locales.
- **61 NgModules, 50 componentes con `standalone: false`.** No bloquea —ya se meten directivas standalone en NgModules— pero es un paso extra.

## Módulos hechos

| Módulo | Alcance | Commit |
|---|---|---|
| `settings/rates` → `ds-rates` | **completo**, cero Material | `db94957e` … `55d2511c` |
| `settings/stations` → `ds-stations` | contenedores + diálogos + snackbar | `d19e3f42` |
| `settings/tipping` → `ds-tipping` | contenedores + controles + snackbar | `0c633145` |

**`ds-rates` está funcionalmente roto** — Israel: «hay muchas cosas que se rompieron». No hay lista todavía. Ver [[project_ds_rates]].

**El alcance no tiene que ser el módulo entero.** Stations y tipping son «contenedores, diálogos y snackbar» por decisión suya, y salió bien: son módulos cortos y el riesgo se acota.

## El orden, por dependencia y no por tamaño

**Fase 0 — en la lib. HECHA.** Ver [[project_corepark_ui_form_fields]].

**Fase 1 — mecánico, ~410 ficheros:** snackbars (224) contra la fachada, form-field + input (~110), iconos a Tabler (68), `MatRipple` → `cpRipple` (6), recortar `MaterialModule`.

**Fase 2 — necesita Fase 0, y ya está desbloqueada:** select/autocomplete/chips (~60), checkbox/switch/radio (~50), menu (35).

**Fase 3 — por módulo, no por lote:** tablas (62+52+26) son el rebuild de cada módulo; diálogos (137 + `wrapper-dialog` con 263 ficheros) van de uno en uno.

**Fase 4 — el remate:** borrar `custom-theme.scss` y los 14 SCSS de overrides, quitar `@angular/material` del `package.json`, unificar botones e iconos.
