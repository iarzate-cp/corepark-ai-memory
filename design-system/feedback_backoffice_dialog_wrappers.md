---
name: Backoffice dialog wrappers — no migrar a cp-dialog-content por lote
description: wrapper-dialog y dialog-wrapper deben mantenerse con su implementación MatDialog; migrar dialog a dialog directamente a DialogService
type: feedback
originSessionId: 68fbc8bf-6233-4778-ae34-196a53bbfd68
---

No migrar `wrapper-dialog` ni `dialog-wrapper` para que usen `cp-dialog-content` internamente. La estrategia correcta es migrar cada dialog individualmente de MatDialog → DialogService + cp-dialog-content directamente.

**Why:** Migrar los wrappers internamente broke visualmente ~60+ dialogs de golpe: sin background, estilos del DS en modo claro vs contenido oscuro, ViewChild undefined. Demasiado blast radius para una sola PR.

**How to apply:** Cuando el usuario pida migrar un dialog, crear un componente nuevo que use `DialogService` + `cp-dialog-content` directamente. No tocar los wrappers hasta que todos los dialogs estén migrados; solo entonces eliminarlos.
