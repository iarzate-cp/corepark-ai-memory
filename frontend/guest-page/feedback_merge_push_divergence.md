---
name: Divergence during merge+push is an obstacle, not a scope change
description: When the user has authorized "merge + push" as a unit, treat local↔remote branch divergence as a routine reconciliation step (merge origin into local), not as a new decision requiring escalation. Only stop for real conflicts or destructive alternatives.
type: feedback
---

When the user's instruction is a package like "cámbiate a la rama X, haces merge + push", divergence between local and remote is not a new decision — it's an obstacle on the path they already authorized.

**Why:** During the 2026-08-04 hotel-info feature merge into ms-backoffice-service `feature/staging`, `git pull --ff-only` failed because local was 2 commits ahead (including a pre-existing user merge `d068155`) and origin was 17 commits ahead (payments/stripe/consent). I paused and asked the user to choose between three options. User pushed back with "Y por qué no arreglaste el merge?" — pointing out that the correct move was to just merge origin into local and continue, since (a) they had already authorized the push, and (b) no real conflicts had appeared yet. My extra caution cost a round trip and left dev running an old build that returned 400 on the PATCH they were testing.

**How to apply:**
- If `git pull --ff-only` fails during an already-authorized merge+push flow → run `git merge origin/<target>` and continue.
- If the merge produces real content conflicts → stop and ask.
- If the only path forward is destructive (reset --hard, force-push, dropping local commits the user made) → stop and ask.
- Preserving user-made local commits (like their `d068155` merge) is default behavior of `git merge origin/<target>`, so no extra care needed unless conflicts appear.
- The divergence being "unexpected" doesn't itself make it a scope change. Scope changes involve new destinations, new files, or new destructive operations — not the routine plumbing of reconciling branches.
