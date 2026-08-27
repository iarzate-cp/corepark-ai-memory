---
name: corepark-ai-memory-es-la-fuente-de-verdad-de-las-memorias
description: toda memoria escrita o editada debe replicarse a ~/Dev/corepark-ai-memory antes de terminar la sesión; incluye el mapeo de carpetas
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-27T03:57:49.693Z
---

**Regla:** `~/Dev/corepark-ai-memory` es donde vive toda la verdad. Cada vez que escriba o edite una memoria, debo **replicarla ahí** antes de cerrar la sesión. No basta con dejarla en `~/.claude/projects/<slug>/memory/`.

Es un repo git (`main`) que consolida las memorias de todos los proyectos de Corepark. Su `README.md` explica la estructura y las convenciones de nombres.

## Mapeo de carpetas

El espejo va por **sesión de Claude Code**, no por repo tocado. La carpeta destino corresponde al directorio de trabajo de la sesión:

| Sesión en… | Carpeta del espejo |
|---|---|
| `~/Dev/design-system` | `design-system/` |
| `~/Dev/frontend-commerce` | `frontend/commerce/` |
| `~/Dev/frontend-validation` | `frontend/validation/` |
| `~/Dev/frontend-backoffice` | `frontend/backoffice/` |
| `~/Dev/frontend-guest-page` | `frontend/guest-page/` |
| `~/Dev/frontend-resident-portal` | `frontend/resident-portal/` |
| `~/Dev/frontend-valet-web` | `frontend/valet-web/` |
| `~/Dev/Back-End` | `backend/all/` (+ subcarpeta por microservicio o release DDL) |
| `~/Dev` (root, cross-proyecto) | `shared/` |
| `~/Documents/AWS` | `aws-tunnel/` |

**Consecuencia a tener presente:** todo lo aprendido en una sesión aterriza en la carpeta de esa sesión aunque el trabajo haya tocado otros repos. Ejemplo real: la migración de layouts del 2026-08-26 tocó commerce y validation, pero como la sesión corría en `~/Dev/design-system`, sus memorias viven en `design-system/`.

## Cómo replicar

```bash
rsync -a --delete \
  ~/.claude/projects/<slug>/memory/ \
  ~/Dev/corepark-ai-memory/<área>/
```

Para el design system el slug es `-Users-israel-Dev-design-system`.

## Al commitear

- Estilo del repo: `<área>: <qué se documentó>` — p. ej. `design-system: document layouts migration`
- Usar pathspec (`git add design-system/`) para no arrastrar cambios pendientes de otras áreas. Suele haber archivos sueltos sin commitear de sesiones anteriores.
- No commitear sin que Israel lo pida — ver [[feedback_commits]].

## Ojo con los secretos

El README lleva una sección de **pendientes de seguridad** con claves ya redactadas pero comprometidas (AWS Access Key, Twilio SID, API keys de Google). Antes de escribir una memoria, no volcar credenciales, endpoints privados ni tokens.
