---
name: Corepark Design System — repo local + package quirks
description: DS source lives in ~/Dev/design-system; consumed as @corepark/corepark-ui from private registry npm.pkg.github.com
type: reference
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
El código fuente del design system vive en `~/Dev/design-system/` (adyacente a este repo). Estructura relevante:

- `projects/corepark-ui/src/lib/components/<name>/` — source + `.types.ts` de cada componente
- `projects/demo/src/app/pages/<name>/` — ejemplos de uso reales (base para copiar patrones)
- `projects/corepark-ui/src/lib/tokens/` — tokens SCSS (`_colors`, `_spacing`, `_radius`, etc.)

**Package quirks:**
- Nombre en npm: `@corepark/corepark-ui` (scope `@corepark`, NO `@corepark-ui` como uno pensaría por el usuario diciéndolo así).
- Registro privado: `https://npm.pkg.github.com`. Requiere:
  - `.npmrc` a nivel proyecto: `@corepark:registry=https://npm.pkg.github.com`
  - Token de GitHub en `~/.npmrc` global (`//npm.pkg.github.com/:_authToken=ghp_...`) — el usuario ya lo tiene.
- Selectores prefijo `cp-*`: `cp-date-range-picker`, `cp-notification`, `cp-table`, etc.
- Estilos globales NO se aplican automáticamente: agregar `node_modules/@corepark/corepark-ui/styles.css` en angular.json `"styles"` para que los tokens `--cp-ui-*` resuelvan (si no, los componentes pierden background, border, shadow).

**How to apply:**
- Antes de reinventar UI, chequear `~/Dev/design-system/projects/corepark-ui/src/lib/components/` — probablemente ya existe.
- Copiar patrón de uso desde `projects/demo/src/app/pages/<name>/*.ts` para ver props, servicios asociados, y proveedores.
- Los `.d.ts` compilados en `node_modules/@corepark/corepark-ui/types/` son truth source si el repo local no está actualizado.
