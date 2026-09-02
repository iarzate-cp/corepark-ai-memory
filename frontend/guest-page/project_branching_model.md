---
name: Branching model — develop is this repo's test branch
description: La regla completa vive en shared/feedback_branching_model.md; lo propio de este repo es que su rama de pruebas se llama develop, no feature/staging
metadata:
  type: project
---
**La regla completa está en `shared/feedback_branching_model.md`** — toda rama sale de `main`, la rama de pruebas absorbe todo, `main` solo entra por PR. No dupliques ese contenido aquí.

Lo único propio de `frontend-guest-page`:

- Su rama de ambiente de pruebas es **`develop`**, no `feature/staging` como en los microservicios.
- `develop` suele tener features que `main` no tiene todavía (a agosto-2026: el hotel-info gate, el custom logo header, la integración de Stripe). Al mergear una rama nacida de `main` hacia `develop`, git resuelve solo en la mayoría de los casos, pero **hay que compilar después del merge** (`npx tsc --noEmit -p tsconfig.app.json` y `npx ng build -c development`): el auto-merge textual puede quedar limpio y aun así dejar el proyecto roto, porque las dos features tocan los mismos archivos (`ticket.component.html/.ts`, `ticket.imports.ts`, `definitions/common.ts`).
