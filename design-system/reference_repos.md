---
name: Repository Paths & Key Files
description: Absolute paths to all three repos and their most important files
type: reference
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
---
## Design System repo

- **Root:** `/Users/israel/Dev/design-system`
- **Library source:** `projects/corepark-ui/src/`
- **Library dist:** `dist/corepark-ui/`
- **npm registry config:** `.npmrc` → `@corepark:registry=https://npm.pkg.github.com`
- **Build + sync commerce:** `npm run build && npm run sync:commerce`
- **Current version:** `0.0.13` (publicada 2026-06-09)
- **Key files:**
  - `projects/corepark-ui/src/styles.scss` — consumer entry point (source)
  - `dist/corepark-ui/styles.scss` — consumer entry point (dist, auto-fixed by build:fix-paths)
  - `projects/corepark-ui/src/lib/components/button/button-directive.ts` — ButtonDirective
  - `projects/corepark-ui/src/lib/tokens/tokens.scss` — token barrel
  - `projects/corepark-ui/src/lib/modules/` — módulos de dashboard (5 completados)

## Frontend Commerce repo

- **Root:** `/Users/israel/Dev/frontend-commerce`
- **App name:** "commerce" — NO confundir con "backoffice" (son repos distintos)
- **Port:** `4400` con SSL (`ng serve --port 4400 --ssl`)
- **App styles:** `src/styles.scss`
- **Installed lib:** `node_modules/@corepark/corepark-ui/` (v0.0.13, fijada sin caret; package manager: pnpm)
- **Valet Dashboard:** `/valet-dashboard` — página con los 5 módulos de la librería
- **Nav config:** `src/app/shared/layouts/main-layout/main-layout.component.ts` — array `menuNav`
- **Nav icon imports:** `src/app/shared/layouts/main-layout/main-layout.config.ts`
- **UI Components showcase:** `src/app/pages/ui-components/` — muestra todos los componentes del DS

## Frontend Validation repo

- **Root:** `/Users/israel/Dev/frontend-validation`
- **App name:** "validation-portal" — portal de validación
- **Branch activo:** `chore/update-angular-version`
- **Port:** `4600` con SSL (`ng serve --port 4600 --ssl`)
- **Angular version:** 20.x (nota: peerDeps del DS son ^21, pero el rsync bypassa el check)
- **Package manager:** pnpm
- **Installed lib:** `node_modules/@corepark/corepark-ui/` (via rsync, no en package.json)
- **Sync command:** `npm run sync:validation` (desde design-system)

## Frontend Backoffice repo

- **Root:** `/Users/israel/Dev/frontend-backoffice`
- **App name:** "backoffice" — NO confundir con "commerce" (son repos distintos)
- **Branch activo:** `develop`
- **App styles:** `src/styles.scss`
- **Angular config:** `angular.json` — uses `@angular-devkit/build-angular:application` (esbuild)
- **SCSS loadPaths:** `src/assets/scss` (en angular.json `stylePreprocessorOptions`)
- **Installed lib:** `node_modules/@corepark/corepark-ui/` (GitHub Packages)
- **Port:** `4100` con SSL
- **Package manager:** pnpm
- **Key notes:**
  - `tsconfig.json` tiene `skipLibCheck: true`
  - corepark-ui styles en `angular.json` styles array: `"node_modules/@corepark/corepark-ui/styles.css"` (PRIMERO en el array)
  - Usa su propio sistema de tokens: `--color-main-*` (teal), `--color-grey-*` — separado de los tokens del DS
  - `data-theme="dark"` NO está en `index.html` — el área de contenido es clara; los tokens DS renderizan modo claro por defecto
  - Dialog wrappers (`wrapper-dialog`, `dialog-wrapper`) usan MatDialog; NO migrados a `cp-dialog-content`
