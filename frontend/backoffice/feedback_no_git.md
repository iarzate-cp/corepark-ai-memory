---
name: Git — nunca por iniciativa propia, pero completo cuando lo piden
description: El default sigue siendo solo editar archivos. Cuando el usuario pide git explícitamente, ejecutarlo completo (commit+push+merge) sin volver a preguntar en cada paso.
type: feedback
originSessionId: 15f1cd0b-724b-4750-8329-9c59d168c9ad
---

**Default:** no crear commits, no crear branches, no hacer push. Terminar en cuanto los archivos estén modificados, y avisar fuerte que quedaron sin commitear (ver [[feedback_flag_uncommitted_edits]]).

**Cuando el usuario lo pide explícitamente, sí se hace — y completo.** Ejemplos textuales: *"hay que crear la rama correspondiente que salga de main"*, *"vamos a hacerle commit + push, nos vamos a feature/staging y hacemos merge + push"*. Eso es autorización para toda la secuencia, no para el primer paso.

**Why:** el usuario controla el flujo de git y no quiere que se le adelanten; pero cuando delega la secuencia completa, preguntar en cada eslabón cuesta un round-trip por paso.

**How to apply:**
- La autorización es **por sesión y por secuencia**, no permanente. La siguiente tarea vuelve al default.
- Una divergencia descubierta a medio flujo es un **obstáculo que se rodea** (típicamente `git merge origin/<target>`), no un cambio de alcance que se escala — salvo que aparezcan conflictos reales. Confirmado dos veces: Hotel Info Gate (2026-08-04) y Ticket Log Room Number (2026-08-13).
- Antes de mergear a una rama compartida: `git fetch` + `git rev-list --left-right --count <local>...<origin>`. Los `feature/staging` se desfasan seguido, en ambas direcciones.
- Verificar (build/tests) **antes** de pushear a una rama compartida, no después.
- La autorización sobre los repos de código **no** se extiende a `corepark-ai-memory` en el mismo aliento. Pero *"documentemos todo"* / *"cerrar el día"* sí cubre commitear y pushear ese repo — es el cierre natural de la instrucción, no un paso extra que haya que confirmar.
