---
name: Rama principal es main, no master
description: La rama principal del repo siempre es main — master lleva años sin usarse y debe ignorarse
type: feedback
originSessionId: b4273c1a-e969-4350-954e-61a3d91558d6
---
La rama principal de este repositorio es `main`, no `master`.

**Why:** `master` lleva años sin actualizarse y está abandonada. Usarla como base o como destino de PRs sería un error.

**How to apply:** Siempre hacer branch off desde `main`, siempre comparar PRs contra `main`, nunca referenciar `master` como rama base ni destino.
