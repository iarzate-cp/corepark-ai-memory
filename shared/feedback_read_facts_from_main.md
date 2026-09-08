---
name: Los datos se sacan de main, no de la rama de pruebas
description: feature/staging contiene trabajo que nunca tocó producción, así que no sirve como fuente de verdad para saber qué existe. Verificar contra origin/main.
metadata:
  type: feedback
---
Cuando haya que averiguar **qué existe hoy** —si una columna se lee, si un endpoint está, si un campo ya viaja— la referencia es `origin/main`, no la rama de pruebas.

`feature/staging` absorbe todo y acumula ramas que pueden no haber llegado nunca a producción. Un working tree parado ahí miente sobre el estado real del sistema.

```bash
git show origin/main:ruta/al/Archivo.java | grep …
git log origin/main..origin/feature/staging --oneline -- ruta/al/archivo
git merge-base --is-ancestor <commit> origin/main && echo "está en prod"
```

**Why:** Israel lo pidió el 2026-09-08 — *"recuerda que en staging hay cosas que no han tocado producción, entonces si quieres sacar datos, que sea de main"*— después de que leí `GuestPageDao` desde un árbol parado en `feature/staging` y reporté como existente un `UPDATE` null-guarded de `expected_departure` que en realidad solo estaba en staging, del hotel info gate sin mergear. Eso cambiaba el alcance real del trabajo.

**How to apply:** antes de afirmar que algo existe, verificarlo contra `origin/main` y decir explícitamente cuándo una pieza es solo de staging. Al planear, asumir que lo de staging **puede** no llegar. Cuidado especial con `git status` recién cambiado de rama: el working tree es de la rama actual, no de la que interesa.

Corolario para merges: un merge de una rama nacida de `main` hacia la rama de pruebas también arrastra los commits de `main` que le faltaban a ésta. Es correcto, pero hay que revisarlo (`git log origin/<destino>..HEAD --oneline`) y avisarlo. Y revisar el auto-merge, no solo los conflictos marcados: en esta sesión Git fusionó campos y columnas **duplicados** sin quejarse, y no compilaba. Ver [[feedback-branching-model]].
