---
name: Flag uncommitted edits loudly when user manages git
description: After editing files, state explicitly that the change is uncommitted so the user doesn't merge/deploy past it
type: feedback
originSessionId: f07718db-6a66-4d7e-a6be-5f3c2a44ce61
---
Cuando aplique un `Edit`/`Write` y el usuario lleva git manualmente (regla `feedback_no_git.md`), terminar la respuesta dejando claro que el cambio sigue en working tree y necesita commit antes de cualquier merge/push/deploy. No basta con decir "ya está corregido" — eso suena a "ya quedó en la historia" y el usuario puede hacer merge/release sin darse cuenta de que el archivo nunca se committeó.

**Why:** El fix de URL del Valet Analytics Report (`pathSetter('/valet-analytics-report')` → `/reports/...`) lo apliqué con `Edit` sobre `develop` y reporté "fixed". El usuario hizo merge de la feature branch a `develop` y desplegó a producción sin el fix — porque mi cambio nunca entró en ningún commit, vivió solo en el working tree. Costó un hotfix posterior + push extra.

**How to apply:** Después de cualquier `Edit`/`Write` cuando aplique la regla de "no git":
1. Cerrar la respuesta con una línea explícita del tipo: *"Cambio aplicado en working tree, sin commitear todavía — recuerda incluirlo en tu próximo commit antes del merge/push."*
2. Si el cambio aplica a una rama distinta a la que se desplegará (ej. edito en `develop` pero la rama de release es `feature/x`), avisar también qué rama necesita recibir el cambio.
3. Si detecto que el usuario está por mergear/deployar (por contexto: "ya hice merge", "voy a subir", "vamos a release"), repetir el aviso aunque ya lo haya dicho antes.
4. Esta regla NO aplica cuando el usuario me autoriza explícitamente a hacer git ("commitea", "haz push", "aplica los ajustes para que llegue a X") — ahí el commit es mi responsabilidad y se cierra el loop yo mismo.
