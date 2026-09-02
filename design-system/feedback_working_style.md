---
name: Working Style Preferences
description: Validated patterns for how to collaborate with Israel — autonomy, scope, communication
type: feedback
originSessionId: 2a38666b-8ed5-4a8a-af67-066ed3e5a108
modified: 2026-08-29T00:14:46.663Z
---
## Give full autonomy when authorized

When Israel says "arregla lo que consideres necesario en el orden que quieras" or similar, proceed with all identified improvements without further check-ins. He trusts architectural judgment and prefers one clean delivery over back-and-forth approvals.

**Why:** He's a senior developer who has already thought through the problem space; micro-confirmations slow him down.
**How to apply:** When the user grants open-ended authorization, execute the full plan, then present a concise summary of what changed and why.

## Terse summaries, no trailing explanations

End responses with a clear status (done/blocked) and what's next — not a recap of what just happened.

**Why:** Confirmed by user acceptance without pushback on concise responses.
**How to apply:** After completing a task, 1–2 sentence wrap-up max. Detailed context goes in commit messages or comments, not in chat.

## Las reglas de CLAUDE.md aplican en las apps consumidoras, no solo en el DS

Israel me corrigió el **2026-08-28** por escribir un `for` con `let` y `unshift` dentro de `frontend-backoffice`. El `CLAUDE.md` vive en el repo del design system, pero sus reglas (`const` siempre, sin mutación, `map/filter/reduce` sobre bucles, sin `any`, `#private`) rigen en los cuatro repos.

**Why:** es un solo producto con un solo estilo de casa; el archivo dice "any code in the Corepark dashboard".

**How to apply:** antes de dar por buena una función nueva en cualquier repo, releerla buscando `for`, `let` y mutación. Recorrer una lista enlazada (`firstChild`) sale más limpio recursivo, y además queda consistente con los helpers que ya existen.

## Spanish in chat, English in code

User writes in Spanish; respond in kind. All file names, variable names, comments, and commit messages remain in English.

**Why:** Natural language preference; code conventions are English.
**How to apply:** Never mix languages within a single medium.
