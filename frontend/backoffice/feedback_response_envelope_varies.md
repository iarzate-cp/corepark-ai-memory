---
name: API response envelope is NOT universal — verify before unwrapping .data
description: CLAUDE.md says { data, code, message } but some services (analytics) spread payload at top level; check actual response first
type: feedback
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
El CLAUDE.md dice "API response shape: `{ data: T, code: string, message: string }` — extract `.data`". Es cierto para la MAYORÍA de endpoints, pero no todos.

**Why:** En Activity by Rate Class asumí el patrón general y escribí `map(({ data }) => data)`. El backend spread el payload al top level: `{ code, message, granularity, buckets }`. Resultado: `data` era `undefined`, el signal quedaba en undefined, y el template crasheaba con `Cannot read properties of undefined (reading 'buckets')`.

El artifact del handoff decía explícitamente: "Todos los endpoints retornan `{ code, message, ...payload }`" — con SPREAD. Yo lo leí pero apliqué el patrón viejo por inercia.

**How to apply:**
- Antes de escribir `map(({ data }) => data)`, verificar la shape real:
  - Si el handoff / artifact tiene ejemplo de response → leer con cuidado si el payload está anidado (`data: { ... }`) o al top level (`granularity: ..., buckets: ...`).
  - Si no hay ejemplo → hacer un fetch real (o pedirle al usuario un curl replay) antes de asumir.
- Tipar el response body como el shape que llega realmente. Si es flat, `HttpClient.post<ResponsePayload>(...)` sin unwrap.
- Si vas a defender contra ambos casos (raro), usar guards: `if (!payload?.buckets?.length) return null` en vez de `payload === null`.
