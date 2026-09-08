---
name: La suite de tests no corre — falta tsconfig.spec.json
description: `ng test` aborta con TS500 porque tsconfig.spec.json se borró en marzo 2025 y no está en main ni en feature/staging, aunque hay 19 archivos .spec.ts vivos. La verificación disponible es el build.
metadata:
  type: project
---
`npm test` en `frontend-guest-page` aborta antes de correr nada:

```
An unhandled exception occurred: error TS500: ENOENT: no such file or directory,
lstat '.../tsconfig.spec.json'
```

El archivo se borró en el commit `240f16d0` ("update version, update dependencies…", 2025-03-11) y **no existe ni en `origin/main` ni en `origin/feature/staging`**. Aun así hay **19 archivos `.spec.ts`** vivos en `src/`, así que el repo parece tener pruebas y no las tiene.

No es algo que rompió un cambio reciente: verificado contra las dos ramas.

**Consecuencia práctica:** en este repo la única verificación automática disponible es `npm run build`, que con `strictTemplates: true` sí cacha bastante —bindings inexistentes, tipos de template— pero no comportamiento. Al pushear a `feature/staging`, que es rama compartida, hay que decirlo explícitamente en vez de reportar "suite verde".

Contrasta con `frontend-commerce`, que usa Jest y sí tiene suite real (57 specs) — aunque ahí falta `jest-environment-jsdom` en `package.json`, así que tampoco corre sin instalarlo.

Arreglarlo es un `chore/` chico y aparte: restaurar `tsconfig.spec.json` y ver cuántos de los 19 specs siguen siendo válidos después de año y medio.
