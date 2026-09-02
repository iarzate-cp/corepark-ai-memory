---
name: Branching model — la rama de pruebas es feature/staging desde sep-2026
description: El pipeline de dev se movió de develop a feature/staging el 2026-09-02; develop ya no despliega. La regla general vive en shared/feedback_branching_model.md
metadata:
  type: project
---
**La regla completa está en `shared/feedback_branching_model.md`** — toda rama sale de `main`, la rama de pruebas absorbe todo, `main` solo entra por PR. No dupliques ese contenido aquí.

Lo propio de `frontend-guest-page`:

- **La rama de pruebas es `feature/staging`**, no `develop`. El pipeline se movió el 2026-09-02 y quedó verificado desplegando dos veces seguidas. Antes de ese día no existía `feature/staging` en el repo.
- **`develop` ya no despliega nada.** Sigue existiendo con historia propia, pero lo que llega a dev es `feature/staging`. Si vas a mergear ahí "para probar", no vas a ver nada.
- Cuando el ambiente de dev "no refleja" un cambio, **verifica el bundle antes de dudar del código**: bajar `main.js` del CloudFront (`dgnw8qm5ijo58.cloudfront.net`), sacar los chunks y grepear un string que solo exista desde tu commit. El 2026-09-02 dev llevaba desde el 25 de agosto sirviendo un build viejo, y no era culpa del buildspec ni del trigger: el pipeline apuntaba a una rama que no existía.
- `develop` cargó durante un mes features que `main` no tenía (hotel-info gate, custom logo, Stripe). Si mergeas algo nacido de `main` hacia una rama que las tenga, **compila después del merge** (`npx tsc --noEmit -p tsconfig.app.json` y `npx ng build -c dev`): el auto-merge textual puede quedar limpio y aun así romper, porque tocan los mismos archivos.
- Para servir local contra la API de dev hay script propio: **`pnpm start:dev`** (`ng serve --configuration=dev --ssl --port 4100`). El host de dev ya tiene `https://localhost:4100` en su CORS.
