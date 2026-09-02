---
name: Repository Paths & Key Files
description: Absolute paths to the four repos, their most important files, and the abbreviations Israel uses for them
type: reference
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
modified: 2026-08-31T20:37:21.613Z
---
## Cómo se refiere Israel a cada repo

Usa abreviaturas de esqueleto consonántico y también el nombre del producto. Anotado el 2026-08-31 tras tener que inferir "sync con CMRS y PRN":

| Dice | Es | Producto | Sync |
|---|---|---|---|
| **BO** | `frontend-backoffice` | BackOffice | `sync:backoffice` |
| **CMRS**, CRM, commerce | `frontend-commerce` | CRM | `sync:commerce` |
| **PRN**, Partner, validation | `frontend-validation` | Partner | `sync:validation` |
| DS, la lib | `design-system` | corepark-ui | — |

Ojo: **el producto y el repo no se llaman igual** en commerce y validation — el repo es `frontend-commerce` pero el producto es "CRM", y `frontend-validation` es "Partner". Ver [[feedback_repo_names]].

`frontend-valet-web` existe pero está **fuera** de la migración por decisión de Israel.

## Design System repo

- **Root:** `/Users/israel/Dev/design-system`
- **Library source:** `projects/corepark-ui/src/`
- **Library dist:** `dist/corepark-ui/`
- **npm registry config:** `.npmrc` → `@corepark:registry=https://npm.pkg.github.com`
- **Build + sync commerce:** `npm run build && npm run sync:commerce`
- **Current version:** `0.0.28` (publicada 2026-08-26) — pero el `dist/` local va por delante, ver [[project_migration_status]]
- **Comandos:** `pnpm run build`, `pnpm run sync:{commerce,validation,backoffice,valet}`, `pnpm run publish:lib`, `pnpm test`
- **Registro:** GitHub Packages, autenticado como `iarzate-cp`
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
- **App name:** "validation-portal" — el producto se llama ahora **Partner** (antes "Hotel Link" / "Validation Portal")
- **Rama de integración:** `develop` (NO `main` — iba 43 commits por delante). Rama activa 2026-08-26: `feature/auth-layout-ds`
- **Port:** `4600` con SSL (`ng serve --port 4600 --ssl`)
- **Angular version:** 21.2.x — los peerDeps `^21` del DS cuadran
- **Package manager:** pnpm (ojo con `minimumReleaseAge`, ver [[feedback_angular_gotchas]])
- **Installed lib:** en `package.json` como `^0.0.28`
- **Sync command:** `pnpm run sync:validation` (desde design-system)
- **Angular project name:** `validate` (los builds salen en `dist/validate`)

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
