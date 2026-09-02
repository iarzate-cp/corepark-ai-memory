---
name: el-repo-design-system-es-solo-libreria-sin-demo
description: projects/demo fue eliminado el 2026-08-27; el repo es un único proyecto Angular library y no hay forma de verificar visualmente los componentes dentro del repo
metadata: 
  node_type: memory
  type: project
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-27T21:29:30.025Z
---

**El demo ya no se usa y fue borrado el 2026-08-27.** `angular.json` tiene ahora **un solo proyecto**: `corepark-ui` (`projectType: library`).

Lo que se fue con él: `projects/demo/` (81 archivos, 2 MB), su bloque en `angular.json`, el script `dev` de `package.json`, y el path `@shared/components/*` de `tsconfig.json`. Se verificó que la librería no importaba nada del demo y que los tests tampoco lo tocan.

## Consecuencias prácticas

- **No hay `pnpm dev`.** El repo no arranca nada. El ciclo de trabajo es `pnpm build` + `pnpm run sync:<app>` (rsync a los `node_modules` del consumidor) y mirar el cambio en la app real.
- **No hay verificación visual dentro del repo.** Los 279 tests cubren lógica, ARIA y estructura del DOM — no cómo se ve nada. Bugs como el del rail `position: fixed` del `cp-app-layout` escapándose de su contenedor solo se ven renderizando. Si hay que revisar algo visual, toca hacerlo en commerce / validation / backoffice tras un sync.
- El presupuesto `anyComponentStyle` de `angular.json` desapareció con el target del demo. El build de la librería (`ng-packagr`) no aplica presupuestos de CSS, así que ya no hay nada que ajustar por ahí.
- El lenguaje de estilos del demo (`.doc-*`, `app-doc-block`, etc.) se fue con la carpeta. Lo que describe [[project_demo_docs_language]] es **histórico** — sirve si algún día se reconstruye un showcase, no como estado actual.

## Builders vigentes

| Target | Builder |
|---|---|
| `build` | `@angular-devkit/build-angular:ng-packagr` |
| `test` | `@analogjs/vitest-angular:test` con `vite.config.mts` (aunque en práctica se corre `pnpm test` → `vitest run` directo) |

`vite` tuvo que declararse como devDep explícita: `vite.config.mts` lo importa y el `node_modules` viejo lo tapaba por hoisting. Un `rm -rf node_modules && pnpm install` lo destapó.

Ver [[project_design_system]] y [[feedback_design_system_patterns]].
