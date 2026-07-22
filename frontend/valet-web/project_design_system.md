---
name: Corepark Design System
description: How @corepark/corepark-ui is set up — published on GitHub Packages, source at /Users/israel/Dev/design-system
type: project
originSessionId: d0941f61-65a6-4871-a21b-3bac650a8857
---
El design system de Corepark es un paquete Angular separado publicado en GitHub Packages.

- **Paquete:** `@corepark/corepark-ui@^0.0.19`
- **Registry:** `https://npm.pkg.github.com` (auth token ya está en `.npmrc`)
- **Fuente local:** `/Users/israel/Dev/design-system/projects/corepark-ui/`
- **Dist local:** `/Users/israel/Dev/design-system/dist/corepark-ui/`
- **Exports:** components, directives, modules, tokens (via `public-api.ts`)

**Instalación en valet-web (hecho 2026-06-15):**
- `package.json`: `"@corepark/corepark-ui": "^0.0.19"` en dependencies
- `tsconfig.json`: no requiere paths entry — TypeScript lo resuelve desde `node_modules` automáticamente
- El `.npmrc` ya tenía el scope `@corepark` apuntando al registry de GitHub

**Why:** El design system está publicado en GitHub Packages, no en npm público. El `.npmrc` ya tenía el token de auth configurado.

**How to apply:** Para importar componentes: `import { CpXxxComponent } from '@corepark/corepark-ui'`. Si se hacen cambios en el design system, hay que publicar una nueva versión y actualizar el semver aquí.
