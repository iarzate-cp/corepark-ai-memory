---
name: corepark-ui-categor-as-de-primer-nivel-y-layouts
description: "lib/ tiene categorías hermanas (components, layouts, services, directives, pipes, utils, tokens, modules); la ubicación física define la categoría"
metadata: 
  node_type: memory
  type: project
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-27T21:38:06.393Z
---

`projects/corepark-ui/src/lib/` tiene **categorías hermanas de primer nivel**:

```
lib/
├── components/   elementos de UI hoja
├── directives/   button/ input/ ripple/ tooltip/  ← poblada de verdad el 2026-08-27
├── layouts/      shells de aplicación  ← creado 2026-08-26
├── modules/      módulos de dashboard
├── services/     injectables del sistema  ← creado 2026-08-26
└── tokens/
```

**`pipes/` y `utils/` se borraron el 2026-08-27.** Solo tenían un `index.ts` con `export {}`, cero usos en los cuatro consumidores, y un subpath público que resolvía a nada. Regla nueva en CLAUDE.md: **una categoría se crea cuando tiene su primer miembro**, no antes.

**Regla (documentada en CLAUDE.md):** la categoría de una carpeta es su **ubicación física**. Los layouts van en `lib/layouts/<name>/`, los pipes en `lib/pipes/<name>/`, los servicios en `lib/services/`. Nunca anidados bajo `lib/components/` y re-exportados desde un barrel hermano.

## lib/layouts/

- `auth-layout/` → `cp-auth-layout` (ver [[project_cp_auth_layout]])
- `app-layout/` → `cp-app-layout` (ver [[project_cp_app_layout]])
- `shell-layout/` → `cp-shell-layout` (movido desde `components/layout/`; migrado a BEM SCSS el 2026-08-27, ya sin Tailwind — ver [[feedback_design_system_patterns]])

## Wiring necesario al crear una categoría

1. `lib/<cat>/index.ts` con los re-exports
2. `src/public-api.ts` → `export * from './lib/<cat>'`
3. `tsconfig.json` paths → `"corepark-ui/<cat>"`
4. `projects/corepark-ui/ng-package.json` → entrada en `assets` si tiene `.scss`
5. `package.json` → `build:fix-paths` reescribe `lib/<cat>/` → `<cat>/`

## Deuda de directivas — RESUELTA el 2026-08-27

Las 4 directivas se movieron **físicamente** a `lib/directives/{button,input,ripple,tooltip}/`. Antes vivían bajo `components/` y solo se re-exportaban desde el barrel hermano; los `index.ts` de origen incluso ya decían *"moved to corepark-ui/directives"* — la intención estaba, el movimiento no.

Lo que hubo que recablear, por si se mueve otra:
1. `styles.scss` → `@use 'lib/directives/button/button-directive'` (y ripple)
2. `package.json` → el sed de `build:fix-paths` necesita `s|lib/directives/|directives/|g`, si no el `dist` queda con rutas rotas
3. `ng-package.json` → entrada en `assets` con `output: './directives'`
4. imports internos: `dialog-content-component`, `form-field-component`, `select-component` y el spec de form-field
5. `TooltipPosition` se extrajo a `components/tooltip/tooltip-types.ts` — lo usan **ambas mitades** (el input de la directiva y el del componente overlay), así que dejarlo en el componente obligaba a la directiva a importar de la otra categoría

Verificado en el `dist`: los 9 `@use` resuelven, `.cp-btn` y sus 8 modificadores compilan, y los `.d.ts` siguen exportando las 5 directivas y `TooltipPosition`.

**Excepción que se queda:** `NavActionDirective` (`[cpNavAction]`) sigue en `components/app-nav/` y se exporta desde el barrel de app-nav. Pertenece a esa feature y sus estilos son globales a propósito para alcanzar el contenido proyectado. No es un barrel mentiroso, así que no viola la regla.

Ver también [[project_design_system]] y [[feedback_design_system_patterns]].
