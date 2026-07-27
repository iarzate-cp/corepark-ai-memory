---
name: corepark-ai-memory es la fuente de verdad para documentación
description: Toda memoria/documentación se escribe en ~/Dev/corepark-ai-memory; el auto-memory local es un mirror
type: feedback
---
`~/Dev/corepark-ai-memory/` es el repo canónico donde vive TODA la memoria de CP (frontend, backend, design-system, aws-tunnel, shared, guest-page, resident-portal, valet-web, validation).

**Why:** Israel diseñó este repo para acumular todo el conocimiento del equipo — arquitectura, decisiones, patrones, bugs históricos, políticas de release, etc. Al ser un repo con remoto en GitHub (`iarzate-cp/corepark-ai-memory`), puede compartirlo con el equipo cuando alguien necesita contexto sin depender de que Israel esté disponible. El auto-memory de Claude Code (en `~/.claude/projects/-Users-israel-Dev-frontend-backoffice/memory/`) es solo un mirror local para que el CLI lo cargue automáticamente cada sesión.

**How to apply:**
- Al crear o actualizar una memoria, escribir SIEMPRE en la ubicación correcta dentro de `~/Dev/corepark-ai-memory/<área>/` primero — es la fuente de verdad.
- Espejarla en `~/.claude/projects/-Users-israel-Dev-frontend-backoffice/memory/` para que la sesión activa la cargue (mismo nombre de archivo, mismo contenido, mismo entry en `MEMORY.md`).
- Áreas actuales dentro del repo: `frontend/backoffice`, `frontend/commerce`, `frontend/guest-page`, `frontend/resident-portal`, `frontend/valet-web`, `frontend/validation`, `backend/`, `design-system/`, `aws-tunnel/`, `shared/`. Escoger la subcarpeta correcta por scope de la memoria (ej. una regla que aplica a todo CP → `shared/`; algo de backoffice → `frontend/backoffice/`).
- Si aparece memoria nueva sin equivalente en corepark-ai-memory, avisar al usuario para que la commitee al repo (regla `feedback_no_git.md`: yo no commiteo).
