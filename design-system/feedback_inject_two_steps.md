---
name: feedback_inject_two_steps
description: nunca encadenar inject(X).prop en un campo de clase; inyectar a un campo privado y derivar en una segunda línea
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 3c304b9d-9ff8-4ae9-aeb6-7e2f0a049f97
  modified: 2026-09-01T21:06:14.054Z
---

**Nunca** encadenar la desreferencia sobre `inject()` en un campo de clase:

```ts
readonly heading = inject(TranslatedTitleStrategy).heading   // ❌ mala práctica
```

Siempre en dos pasos — inyectar a un campo privado y derivar después:

```ts
readonly #translatedTitleStrategy = inject(TranslatedTitleStrategy)

readonly heading = this.#translatedTitleStrategy.heading      // ✅
```

**Why:** la forma encadenada esconde la dependencia — no aparece en el bloque de inyecciones, así que leyendo la cabecera de la clase no se ve de qué depende. Y en cuanto hace falta un segundo miembro del mismo servicio acabas con dos `inject()` del mismo tipo o con un refactor.

**How to apply:** el campo privado va **con las demás inyecciones, arriba de la clase** (orden de miembros del CLAUDE.md: inyecciones → inputs/outputs → señales → hooks → métodos), no junto al derivado.

Corregido el 2026-09-01 en diez sitios del BO y validation (`design-switch-overlay`, `wrapper`, `page-meta-row`, `design-shell`, `design-auth-shell`, las tres páginas de auth del BO, y sign-in de validation).

**No aplica a las funciones guard**, que son una sola expresión y no tienen bloque de inyecciones: `export const newDesignGuard = () => inject(DesignState).isNew()` se queda así.

Ver [[feedback_design_system_patterns]].
