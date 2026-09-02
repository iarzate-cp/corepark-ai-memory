---
name: lockfiles-el-gitignore-estaba-al-reves
description: el .gitignore ignoraba pnpm-lock.yaml y trackeaba package-lock.json; corregido el 2026-08-27
metadata: 
  node_type: memory
  type: feedback
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-27T21:38:15.830Z
---

En `design-system` el `.gitignore` **ignoraba `pnpm-lock.yaml`** y en cambio `package-lock.json` estaba **trackeado** (550 KB, obsoleto — todavía listaba Tailwind). Exactamente al revés de la regla del proyecto ([[feedback_package_manager]]).

Corregido el **2026-08-27**: `pnpm-lock.yaml` sale del ignore y se versiona; `package-lock.json` borrado y destrackeado, y añadido al `.gitignore` con un comentario que explica que si reaparece es que alguien corrió `npm` por error.

**Why:** con el lockfile de pnpm sin versionar, cada `pnpm install` podía resolver versiones distintas — no había instalación reproducible para nadie más ni para CI. Y el `package-lock.json` commiteado invitaba a que alguien corriera `npm install` y generara un árbol paralelo.
**How to apply:** revisar esto mismo en los otros repos antes de asumir que su lockfile es fiable. El lockfile de pnpm **se versiona siempre**; el de npm no debe existir.
