---
name: estado-de-la-migraci-n-de-layouts-al-ds-2026-08-26
description: "qué está hecho y qué falta por repo, ramas activas, y el riesgo del build local por delante del registro"
metadata: 
  node_type: memory
  type: project
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-27T03:54:38.827Z
---

Sesión del **2026-08-26**. Nada commiteado en ningún repo.

## design-system — rama `develop`

Versión `0.0.28` **publicada** en GitHub Packages. Pero el `dist/` local y los `node_modules` de las apps tienen un build **posterior** con: gráficos oscuros del auth, `--decor-alt`, transiciones `@property`, `cp-app-layout`, `cp-app-nav`, `cp-app-nav-footer`, `NavActionDirective`, `ColorSchemeService`, `provideColorScheme`, tokens `_breakpoints`/`_transitions`, y todos los ajustes finos del menú.

⚠️ **RIESGO:** commerce y validation declaran `^0.0.28` en `package.json` pero su `node_modules` lleva el build local. **Un `pnpm install` limpio los revierte al registro y se pierde todo.** Publicar `0.0.29` cuando se cierre el diseño.

Sin commitear: `layouts/`, `services/`, `components/app-nav/`, tokens nuevos, fix del isotipo, CLAUDE.md, demo pages (`app-layout`, `auth-layout`), `publish:lib`, scripts a pnpm.

## frontend-validation — rama `feature/auth-layout-ds` (desde `develop`, NO main)

`develop` iba 43 commits por delante de `main`. **Hecho:**
- `/auth` → `DsAuthLayout`; sign-in migrado a `cp-form-field`/`cpInput`/`cpButton`/tabler
- `/` → `DsAppLayout` con logo, `cp-app-nav` (6 links planos), footer con Chat + tema + Sign out
- **Puente de tokens en `root.scss`**: los ~40 tokens locales aliasan a `--cp-ui-*`; los 303 usos en 42 archivos no se tocaron. Se arreglaron 8 tokens que se usaban sin estar definidos
- `body` a `--cp-ui-color-bg-app`; `_custom-table.scss` y `_input-searcher.scss` desharcodeados
- **Tema oscuro de Material** bajo `[data-theme='dark']` con `mat.all-component-colors`
- `provideColorScheme()` del DS registrado
- `product="Partner"`

**Pendiente:** borrar `main-layout` y `auth-layout` viejos (ya no ruteados); los 3 fondos negros hardcodeados de `main-layout/ui/`; los `@font-face` muertos de Poppins/Poltawski.

## frontend-commerce — rama `feature/auth-layout-ds`

**Hecho:** `/auth` → `DsAuthLayout`; `/` → `DsAppLayout` con `product="CRM"`, menú de 1 link + 7 grupos sin ruta propia, footer con OTP + tema + Sign out; sign-in migrado a componentes del DS; `one-time-password` re-pielado con `cpNavAction`.

**Usa su `ColorSchemeState` propio**, NO el `ColorSchemeService` del DS — commerce ya tiene su app initializer y dos dueños del atributo `data-theme` se pelearían. Unificar es un paso pendiente (toca `valet-layout` y `auth-layout`).

**Pendiente:** `.npmrc` con un **PAT de GitHub en texto plano y trackeado en git** — rotar y sacar del control de versiones.

## frontend-backoffice — sin tocar

Tiene `0.0.25`. Es el origen de `cp-app-layout` y `cp-app-nav`. Cuando migre: subir versión, cambiar a Roboto Flex, y decidir qué hacer con `operator-logo` (marca blanca, 12 logos) y `app-nav-mobile` (manipula el DOM a mano).

## Tipografía — decisión tomada

**Todo a Roboto Flex**, la familia del DS. Es variable (`wght@100..900` continuo), así que los 5 tokens de peso se renderizan exactos — Lato no tiene 500/600, Inter no tiene 100/900.

Commerce y validation ya cargan `family=Roboto+Flex:opsz,wght@8..144,100..900` y su reset usa `font-family: var(--cp-ui-font-family)`. Ninguna app declara familia propia. **Falta el backoffice.**

Ver [[project_cp_app_layout]], [[project_cp_auth_layout]], [[feedback_angular_gotchas]].
