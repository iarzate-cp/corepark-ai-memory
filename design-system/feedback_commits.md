---
name: Commit scope — design system only
description: Only commit in the design system repo; never commit in the backoffice
type: feedback
originSessionId: e2987758-e815-4537-8e0c-4daea1db55e9
---
Solo hacer commits en el design system (`/Users/israel/Dev/design-system`). En el backoffice (`/Users/israel/Dev/frontend-backoffice`) no commitear — Israel lo maneja él mismo.

**Why:** El backoffice está en etapa experimental; los commits del DS son los que importa versionar.

**How to apply:** Después de un build + rsync, solo hacer `git commit` en el design system. Nunca en el backoffice.
