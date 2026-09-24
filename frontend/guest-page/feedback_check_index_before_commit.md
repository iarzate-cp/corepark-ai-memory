---
name: feedback_check_index_before_commit
description: "Revisar `git diff --cached` antes de cada commit en este repo, porque el índice suele traer cambios preparados de antes de la sesión"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: a1632e92-0e98-42a8-bf7c-9a40c171e518
  modified: 2026-09-21T22:36:22.865Z
---

Antes de cada `git commit`, mirar `git diff --cached --name-status`. No basta con saber qué archivos añadí yo.

**Por qué:** el 2026-09-21, al partir el trabajo en tres commits, el índice ya traía renames preparados (`git mv` de `stripe-tipping` → `payment-tipping`) desde antes de que empezara la sesión. `git add` de los archivos de pnpm y luego `git commit` se llevó también esos renames, porque **commit versiona todo el índice, no solo lo que acabas de añadir**. Hubo que deshacerlo con `reset --mixed` y rehacer los tres.

**Cómo aplicarlo:** al inicio, `git status --short` delata un índice sucio (columna izquierda con `R`, `A` o `M`). Si va a haber commits atómicos, vaciarlo con `git reset` antes de empezar a preparar el primero. Y cuidado con `git add -u`, que arrastra todos los modificados rastreados, no solo los del grupo que estás armando.
