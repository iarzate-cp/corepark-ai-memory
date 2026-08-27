---
name: corepark-ui-categor-as-de-primer-nivel-y-layouts
description: "lib/ tiene categorías hermanas (components, layouts, services, directives, pipes, utils, tokens, modules); la ubicación física define la categoría"
metadata: 
  node_type: memory
  type: project
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-27T03:53:08.598Z
---

`projects/corepark-ui/src/lib/` tiene **categorías hermanas de primer nivel**:

```
lib/
├── components/   elementos de UI hoja
├── directives/
├── layouts/      shells de aplicación  ← creado 2026-08-26
├── modules/      módulos de dashboard
├── pipes/
├── services/     injectables del sistema  ← creado 2026-08-26
├── tokens/
└── utils/
```

**Regla (documentada en CLAUDE.md):** la categoría de una carpeta es su **ubicación física**. Los layouts van en `lib/layouts/<name>/`, los pipes en `lib/pipes/<name>/`, los servicios en `lib/services/`. Nunca anidados bajo `lib/components/` y re-exportados desde un barrel hermano.

## lib/layouts/

- `auth-layout/` → `cp-auth-layout` (ver [[project_cp_auth_layout]])
- `app-layout/` → `cp-app-layout` (ver [[project_cp_app_layout]])
- `shell-layout/` → `cp-shell-layout` (movido desde `components/layout/`; **sigue con clases Tailwind**, deuda pendiente)

## Wiring necesario al crear una categoría

1. `lib/<cat>/index.ts` con los re-exports
2. `src/public-api.ts` → `export * from './lib/<cat>'`
3. `tsconfig.json` paths → `"corepark-ui/<cat>"`
4. `projects/corepark-ui/ng-package.json` → entrada en `assets` si tiene `.scss`
5. `package.json` → `build:fix-paths` reescribe `lib/<cat>/` → `<cat>/`

**Ojo:** `lib/pipes/index.ts` y `lib/utils/index.ts` estaban vacíos (solo comentario) y no eran módulos válidos. Llevan `export {}` para poder exportarlos desde `public-api`.

**Deuda conocida:** `lib/directives/index.ts` **no contiene ninguna directiva** — re-exporta 4 que viven físicamente bajo `components/` (`ButtonDirective`, `InputDirective`, `RippleDirective`, `TooltipDirective`). Es exactamente el antipatrón que la regla prohíbe. Moverlas es caro porque 3 tienen `.scss` referenciados por ruta en `styles.scss`.

Ver también [[project_design_system]] y [[feedback_design_system_patterns]].
