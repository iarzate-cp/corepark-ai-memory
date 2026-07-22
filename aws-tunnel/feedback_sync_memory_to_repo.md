---
name: Sync memory writes to corepark-ai-memory repo
description: Whenever a memory file is created or updated in ~/.claude/projects/*/memory/, mirror the change into ~/Dev/corepark-ai-memory/ and commit+push.
type: feedback
originSessionId: a6fa95ac-e208-4b34-8192-d815dbe67c78
---
Cada vez que escribas o edites un archivo bajo `~/.claude/projects/<slug>/memory/`
(incluido `MEMORY.md`), replica el mismo cambio en la carpeta correspondiente de
`~/Dev/corepark-ai-memory/`, haz `git add`, commit y push a `origin main`.

**Why:** El repo privado `github.com:iarzate-cp/corepark-ai-memory` es el backup
off-machine y la única forma de que las notas sobrevivan a un reinstalar/cambio
de máquina y de que sean accesibles desde otras sesiones. El usuario pidió
explícitamente que se mantuviera en sync tras el import inicial de 2026-07-22.

**How to apply:**

1. **Mapeo** (según README del repo):
   - `~/.claude/projects/-Users-israel-Dev-Back-End/memory/` → `backend/all/`
   - `~/.claude/projects/-Users-israel-Dev-Back-End-ms-backoffice-service/memory/` → `backend/ms-backoffice-service/`
   - `~/.claude/projects/-Users-israel-Dev-Back-End-ddl-DDL-2026-07-15-COUPONS/memory/` → `backend/ddl-coupons/`
   - `~/.claude/projects/-Users-israel-Dev-frontend-<name>/memory/` → `frontend/<name>/`
   - `~/.claude/projects/-Users-israel-Dev-design-system/memory/` → `design-system/`
   - `~/.claude/projects/-Users-israel-Dev/memory/` → `shared/`
   - `~/.claude/projects/-Users-israel-Documents-AWS/memory/` → `aws-tunnel/`
   - Cualquier nueva `~/.claude/projects/<slug>/memory/` → crear carpeta nueva
     en el repo y añadir línea a la sección "Estructura" del `README.md`.

2. **Security scan antes del commit** (el import inicial se bloqueó por Push
   Protection de GitHub al detectar un Twilio SID que mi scan no cazó):
   ```bash
   grep -rinE "AKIA|ASIA|AC[a-f0-9]{32}|SK[a-f0-9]{32}|(sk|pk|rk)_(live|test)_|AIza[0-9A-Za-z_-]{30,}|gh[pousr]_|xox[baprs]-|BEGIN.*KEY" <path>
   ```
   Si aparece algo real, **redactar antes de commit** (reemplazar el valor por
   `<REDACTED — véase 1Password / consola>`) y actualizar los pendientes de
   seguridad del `README.md` del repo.

3. **Git**:
   - Remote correcto: `git@github.com-work:iarzate-cp/corepark-ai-memory.git`
     (usa el alias `github.com-work` del `~/.ssh/config` que apunta a
     `~/.ssh/id_corepark`). Con `github.com` sin alias falla con
     "Permission denied (publickey)".
   - Commit del usuario: `Israel Arzate <israel.arzate@corepark.com>`.
   - Mensaje breve describiendo qué memoria se tocó, ej.
     `Update ms-backoffice-service: project_coupon_progress_snapshot`.

4. **Pedir OK antes del push**. El push es visible en GitHub y es acción
   con blast radius — confirmar con el usuario antes de `git push`.
