---
name: package-manager-siempre-pnpm
description: "Usar pnpm para todo (install, run, test, sync, publish) — nunca npm, en ningún repo"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-26T20:58:03.769Z
---

Usar **siempre `pnpm`**, nunca `npm`, en todos los repos: design-system, frontend-commerce, frontend-backoffice, frontend-validation.

Aplica a todo: `pnpm install`, `pnpm add`, `pnpm run build`, `pnpm test`, `pnpm run sync:commerce`, `pnpm run publish:lib`.

**Why:** todos los repos usan pnpm como package manager. Mezclar npm genera lockfiles inconsistentes (el design-system tiene `package-lock.json` y `pnpm-lock.yaml` conviviendo, residuo de esa mezcla) y puede reescribir el árbol de `node_modules` de forma distinta a la que espera pnpm.

**How to apply:** sustituir cualquier `npm <cmd>` por `pnpm <cmd>` antes de ejecutarlo. Ojo con los scripts del `package.json` del design-system que se invocan entre sí — ver [[project_design_system]] y [[feedback_design_system_patterns]].
