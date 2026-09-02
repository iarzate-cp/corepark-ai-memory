---
name: commit-scope-d-nde-se-commitea-y-qui-n-decide-publicar
description: se commitea en los cuatro repos; publicar la lib lo pide Israel explícitamente y entonces sí se hace de punta a punta
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 3c304b9d-9ff8-4ae9-aeb6-7e2f0a049f97
  modified: 2026-09-01T23:04:42.760Z
---

**Se commitea en cualquiera de los repos donde se trabaje** — design-system, backoffice, commerce, validation. La nota anterior decía "solo en el design system"; quedó obsoleta el 2026-08-31, cuando los tres consumidores recibieron commits firmados, y se confirmó el 2026-09-01 con una jornada entera de commits en el backoffice.

**No hacer `push` por iniciativa propia.** Commitear sí; empujar, cuando Israel lo pida.

## Publicar la lib

**No proponerlo al cerrar una tarea.** El ritmo de release lo marca él: el trabajo pasa por muchas iteraciones antes de que una versión valga la pena, y ofrecerlo en cada cierre es ruido.

**Cuando lo pida, la secuencia completa** (hecha el 2026-09-01 para `0.0.30`):

1. Bump en `projects/corepark-ui/package.json`.
2. `pnpm build` → verificar que el `dist` lleva la versión y **la API nueva**.
3. `pnpm test` (los 311 tienen que pasar).
4. `pnpm run publish:lib`.
5. Commit del bump en el DS.
6. En el consumidor: `pnpm add @corepark/corepark-ui@X.Y.Z` — **fijada, sin caret**.
7. **Build del consumidor contra el tarball publicado**, no contra el rsync. Es la única verificación de que lo publicado basta.
8. Bump CalVer de la app con `pnpm run release`.

**Por qué importa el paso 7:** durante la jornada los módulos corrían contra un build sincronizado a mano, y tres ya dependían de API que no existía en la versión publicada. Un `pnpm install` limpio los habría dejado sin compilar. Ver [[feedback_pnpm_add_borra_el_sync]].

**El flujo diario sigue siendo** `pnpm build` + `pnpm run sync:<app>`: solo toca `node_modules` y es reversible. Eso sí se propone y se hace.

Ver [[feedback_branching_staging]].
