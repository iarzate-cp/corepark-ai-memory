---
name: Branching model — always branch from main, test envs absorb anything
description: En todos los repos de CorePark toda rama nueva sale de main; feature/staging (o develop) es el ambiente de pruebas y ahí se puede meter de todo; main es producción y solo entra por PR
metadata:
  type: feedback
---
Aplica a **todos** los repos de CorePark — microservicios Java y frontends por igual. Tenerlo presente **al empezar cualquier trabajo nuevo**, antes de crear la rama.

## 1. Toda rama sale de `main`

Sin excepción. `main` es producción y es la única base válida. La secuencia al arrancar algo:

```bash
git switch main && git pull
git switch -c feature/<short-desc>
```

Prefijos: `feature/` · `fix/` · `chore/` · `docs/`, en minúsculas y kebab-case.

**Consecuencia que hay que anticipar, no descubrir a medio merge:** las ramas de prueba tienen commits propios que `main` no tiene, y `main` tiene commits que ellas no tienen. Al mergear una rama nacida de `main` hacia la rama de prueba, viajan también los commits de `main` que faltaban ahí. Eso es correcto y deseable — la rama de prueba no debería correr sin un fix que ya está en producción — pero conviene revisar qué entra (`git log origin/<destino>..HEAD --oneline` y `git diff origin/<destino> --stat`) y avisarlo, porque no es parte del cambio propio.

## 2. La rama de ambiente de pruebas absorbe todo

**Ahí se puede meter de todo sin problema — es exactamente para eso.** CI/CD despliega esa rama al ambiente de test en AWS. Divergencias, non-fast-forwards y merge commits son normales: se reconcilian y se pushea, sin escalar. Ver [[feedback_merge_push_divergence]].

Cuál es esa rama **depende del repo** — verificar en su `CLAUDE.md` o en el pipeline:

| Repo | Rama de pruebas |
|---|---|
| `ms-*` (todos los microservicios) | `feature/staging` |
| `frontend-guest-page` | `develop` |
| `frontend-receipt` | `feature/staging` (pipeline `DEV_frontend-receipt`) |

Aun así conviene compilar y correr la suite antes de pushear: son ramas **compartidas**, y romperlas cuesta tiempo del equipo, no solo del que pushea.

## 3. `main` solo entra por PR

**Nunca** mergear ni pushear a `main` directo, aunque el usuario diga "solo súbelo". El camino es: abrir PR en GitHub (`gh pr create`) → esperar autorización de un reviewer → el reviewer mergea desde la UI. Sin self-merges, sin merges por CLI, sin saltarse la revisión — ni para cambios mínimos.

Si el destino es ambiguo, preguntar. El default es la rama de pruebas, nunca `main`.

## Relacionado

- [[feedback_commit_workflow]] — el default es dar solo el texto del mensaje y no correr `git commit`; el usuario lo sobreescribe por tarea cuando quiere que yo commitee y pushee.
- [[feedback_merge_push_divergence]] — la divergencia al mergear es un obstáculo a resolver, no un cambio de alcance.
