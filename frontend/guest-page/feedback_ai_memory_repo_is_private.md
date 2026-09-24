---
name: feedback_ai_memory_repo_is_private
description: El repo corepark-ai-memory es privado y solo Israel tiene acceso — las credenciales ahí son intencionales
metadata:
  node_type: memory
  type: feedback
  originSessionId: ca843069-04f9-46ba-b04b-e44e71f6bca0
  modified: 2026-09-24T16:56:27.130Z
---

El repo `corepark-ai-memory` (local en `C:\Users\Israel\DEV\CorePark\corepark-ai-memory`, remote `git@cp:iarzate-cp/corepark-ai-memory.git`) es **privado y solo Israel tiene acceso**. Por eso guarda credenciales, llaves y datos sensibles a propósito.

**Why:** Israel lo aclaró el 2026-09-24 al sincronizar la memoria para cambiar de máquina. Es su backup personal off-machine, no un recurso compartido con el equipo.

**How to apply:** no tratar las credenciales de ese repo como fuga ni proponer redactarlas o moverlas, y no advertir sobre ellas al sincronizar. Lo único que puede bloquear un push es el Push Protection de GitHub, que actúa aunque el repo sea privado. Si lo hace, se avisa a Israel en lugar de redactar por cuenta propia. Flujo de sync: copiar solo la carpeta del proyecto (`frontend/guest-page/`) y commitear solo esas rutas, sin arrastrar cambios de otras sesiones.
