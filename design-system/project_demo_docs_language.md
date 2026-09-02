---
name: lenguaje-de-estilos-del-demo-y-componentes-de-documentacion
description: "las clases .doc-* globales del demo, los 4 componentes que absorben la repetición de las 17 páginas, y el truco de especificidad para previsualizar layouts fixed"
metadata: 
  node_type: memory
  type: project
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-27T21:29:39.127Z
---

> ⚠️ **HISTÓRICO.** `projects/demo` fue **borrado** el 2026-08-27, horas después de escribir esto — ver [[project_repo_is_library_only]]. Nada de aquí existe hoy en el repo. Se conserva porque el truco de especificidad y la estructura sirven si algún día se reconstruye un showcase.

Creado el **2026-08-27** al quitar Tailwind. El demo documentaba el DS **con el DS**.

## Capas

1. **`projects/demo/src/styles/_reset.scss`** — repone el preflight de Tailwind.
2. **`projects/demo/src/styles/_docs.scss`** — lenguaje global del showcase. Global a propósito: los templates de página usan estas clases directo y no hay consumidor al que filtrar.
   - Shell: `.doc-page`, `.doc-page--wide` (56rem / 64rem vía `--doc-page-width`)
   - Prosa: `.doc-lede`, `.doc-note` (`--muted`, `--flush`), `.doc-overline`, `.doc-subheading`
   - Chips: `.doc-selector`, `.doc-selectors`, `.doc-title-row`, `.doc-chip`
   - Layout de ejemplos: `.doc-stack` (`--loose`), `.doc-row` (`--tight`, `--wrap`), `.doc-form-col`, `.doc-fill`, `.doc-divider`
   - `.doc-page :is(p, li, .doc-lede, .doc-note) code` — el `<code>` inline de la prosa no necesita clase (mató 42 usos)
3. **SCSS por página** vía `styleUrl` para el residuo específico (`.tabs-page__*`, `.charts-page__canvas`, etc.)

## Componentes compartidos (`projects/demo/src/shared/components/`)

| Componente | Rol |
|---|---|
| `app-doc-page-header` | h1 + chips de selector + lede. Input `selector` acepta `string` **o** `readonly string[]` (input-text documenta `cp-form-field` + `cpInput`) |
| `app-doc-block` | Una banda titulada: regla arriba, `.doc-overline`, contenido. `:host(:last-child)` quita el margen inferior |
| `app-doc-section` | Un ejemplo: cabecera + canvas con retícula de puntos + toggle "ver código" |
| `app-api-table` | Tabla de 4 columnas. Input `nameHeading` (default `'Input'`) para reetiquetar la primera y reusarla en Outputs / Config |
| `app-slot-table` | Hermana de 2 columnas para slots de proyección |
| `app-code-block` | Oscuro en ambos temas a propósito (un sample de código se lee como terminal). Sus 5 grises son el único hex hardcodeado del demo. `.code-block--embedded` para cuando vive dentro de un `app-doc-section` |

## El truco de especificidad para previsualizar layouts

`cp-app-layout` y `cp-auth-layout` se anclan al viewport, así que no se pueden mostrar fieles dentro de una página de docs. Dos trampas distintas:

- **`.doc-preview.doc-preview--app-layout`** — la clase **va duplicada**. Las reglas del layout son encapsuladas (`.cp-app-layout__aside[_ngcontent-x]` = 0,2,0) y su hoja se inyecta cuando carga la ruta lazy, o sea **después** del sheet global. A igual especificidad gana el orden de fuente, así que con una sola clase el rail `position: fixed` se escapa y **tapa el sidebar real del demo**. 0,3,0 gana sin `!important`.
- **`.auth-layout-page__preview cp-auth-layout { --cp-ui-auth-layout-height: 30rem }`** — el knob tiene que aterrizar en `cp-auth-layout`, **no** en el wrapper: el `:host` del layout declara la variable directamente, y una declaración propia le gana a una heredada por muy alto que la pongas.

## Presupuesto de CSS en angular.json

`anyComponentStyle` subió de 4kB/8kB a **12kB/16kB**. Estaba calibrado para una era en que los componentes no tenían SCSS; con BEM en todo, `auth-layout` (8.9kB) y `date-range-picker` (10.3kB) rompían el build de producción — de hecho ya lo rompían antes.

Ver [[feedback_design_system_patterns]] y [[project_corepark_ui_components]].
