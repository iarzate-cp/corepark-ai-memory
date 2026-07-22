---
name: Preferir Array.prototype.at() sobre indexado con corchetes
description: En este proyecto usar `arr.at(0)` en vez de `arr[0]` para acceder al primer elemento (y en general para índices literales)
type: feedback
originSessionId: 9730d6eb-0dfc-4a74-85d0-0ee58183c514
---
Usar `arr.at(0)` en vez de `arr[0]` cuando se accede a un elemento por índice y el array puede estar vacío.

**Why:** El proyecto tiene `strict: false` y no activa `noUncheckedIndexedAccess` (`tsconfig.json:7`), así que `arr[0]` se tipa como `T` y miente cuando el array está vacío. `arr.at(0)` siempre se tipa como `T | undefined`, permite que TypeScript haga narrow correctamente con guards posteriores y evita `undefined` silenciosos en producción.

**How to apply:** Aplica a cualquier acceso por índice en el código nuevo. No aplica si el array es literal y sabes por construcción que tiene ese elemento (ej. destructuring `const [first] = ...` sigue siendo idiomático). Si el índice es negativo (último elemento), `.at(-1)` es el único camino correcto de todas formas.
