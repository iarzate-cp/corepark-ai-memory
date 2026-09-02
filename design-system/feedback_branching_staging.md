---
name: feedback_branching_staging
description: "feature/staging es entorno de pruebas: nunca desprender una rama de ahí ni mergearla a otra; es sumidero, no fuente"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 02d8be60-b4d7-4d73-99ae-4bda6564aa7f
  modified: 2026-08-31T22:06:03.426Z
---

Regla que Israel tuvo que corregirme el **2026-08-31**:

> *"Staging es nuestro ambiente de pruebas, nosotros NUNCA tenemos que desprender ninguna rama de ahí y tampoco tenemos que hacerle merge a alguna otra rama."*

**`feature/staging` es un sumidero, no una fuente.** El trabajo entra para probarse y no sale nunca.

- ❌ `git checkout -b algo feature/staging`
- ❌ `git merge feature/staging` estando en otra rama
- ✅ `git merge <feature>` estando **en** staging

**Why:** staging acumula todo lo que está a prueba, incluido trabajo que puede no llegar nunca a producción. Sacar algo de ahí arrastra features de terceros a medio validar.

**How to apply:** la referencia de "qué debería estar en producción" es **`main`**, nunca staging.

## El error concreto que cometí

Vi que `feature/design-toggle` (la rama que va a prod) le faltaban 30 commits que sí estaban en staging — entre ellos la feature `vehicle-info-links` entera — y propuse `merge feature/staging → feature/design-toggle` para "no perder trabajo". Dos veces.

Estaba midiendo contra la referencia equivocada. Al comprobarlo contra `main`:

- `vehicle-info-links` **no está en `main`** → es trabajo en pruebas, no debe estar en prod.
- `feature/design-toggle` desciende de `main` y **no le falta nada de ahí** (0 commits).

O sea que la ausencia de esos 30 commits era **correcta**, no un bug. La rama de prod estaba completa.

**La lección:** antes de alarmarse por commits ausentes, comprobar contra qué se está comparando. `git rev-list --count <rama>..main` responde la pregunta útil; `..staging` responde una que no importa.

## Consecuencia práctica al arreglar algo en staging

Un fix commiteado **solo** en staging no llega nunca a ninguna parte. Me pasó: arreglé `vehicle-info-links` ahí (necesitaba un opt-out de la regla de headers) y ese commit quedó en un sumidero — cuando la feature llegue a `main` desde su propia rama, el fix no viene con ella.

La salida no fue mover el fix, sino **cambiar el default para que no hiciera falta** — ver [[project_design_toggle_commerce]]. Si un fix solo tiene sentido en staging, es señal de que el diseño depende de que alguien recuerde algo.

Ver [[feedback_commits]] y [[reference_repos]].
