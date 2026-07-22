---
name: Patrón de inject — siempre field separado
description: No encadenar .property directamente en inject(); siempre inyectar en un campo privado primero
type: feedback
originSessionId: 8f161420-0860-4a92-a4cd-a5269c24d5a4
---
No hacer esto:
```typescript
protected readonly loading = inject(SomeState).loading
```

Siempre así:
```typescript
readonly #someState = inject(SomeState)
protected readonly loading = this.#someState.loading
```

**Why:** Preferencia explícita del usuario — más legible, consistente con el CLAUDE.md que indica `#private` fields para injections, y facilita acceder a múltiples propiedades del mismo servicio sin repetir `inject()`.

**How to apply:** En cualquier componente, guard, o state donde se use `inject()` — siempre asignar a un campo `readonly #name` primero.
