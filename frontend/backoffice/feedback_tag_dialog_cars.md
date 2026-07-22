---
name: Tag dialog — cars vienen del padre, no del dialog
description: No mover el fetch de cars adentro del GuestProfileTagDialog; el padre los pasa via TagDialogData
type: feedback
originSessionId: 376a91b8-fb58-42a0-be23-e2e4e071f72b
---
No cambiar `TagDialogData` para quitar `cars: Car[]` ni hacer que el dialog llame `getCars(profileId)` por su cuenta.

**Why:** Se intentó en sesión de mayo 2026 — el dialog hacía `#loadCars()` en `ngOnInit`, convertía `cars` a signal, y eliminaba `cars` del data. El usuario lo revirtió completo: "no necesitábamos nada más".

**How to apply:** Si el dialog necesita la lista de vehículos, el caller (profiling table o cars component) la pasa como `data.cars`. El dialog la usa directamente. No introducir async fetch dentro del dialog para esto.
