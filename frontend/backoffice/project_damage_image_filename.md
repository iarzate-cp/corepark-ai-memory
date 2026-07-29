---
name: Damage photos — convención del filename en S3
description: Nombre `IMG_{ticket}_yyyy-MM-dd-HH-mm-ss-SSS` — contiene el timestamp de captura en hora local del dispositivo del valet
type: project
---
Las fotos de damage vienen de S3 con este formato de filename: `IMG_{ticket}_yyyy-MM-dd-HH-mm-ss-SSS.JPG` (ej. `IMG_1002_2026-07-24-16-09-09-280.JPG`). El key completo en S3 es `companyId/locationId/ticket/checkInAtUtc/filename`.

El timestamp embebido en el filename es la hora **local del dispositivo del valet** (sin zona horaria), no UTC. Es la hora "real" en que se tomó la foto — a diferencia del `lastModified` de S3, que es UTC y refleja cuándo se subió al bucket (puede diferir si hubo retry de red).

**Why:** En el detalle del ticket log (dialog de damage), se mostraba el filename crudo como label bajo la foto — ilegible. En julio 2026 se creó `core/pipes/damage-image-date-pipe.ts` que extrae el sufijo con regex `/(\d{4}-\d{2}-\d{2}-\d{2}-\d{2}-\d{2}-\d{3})$/`, lo parsea con `DateTime.fromFormat(..., 'yyyy-MM-dd-HH-mm-ss-SSS')`, y lo formatea como `MMM dd, yyyy HH:mm:ss a`. Acepta un segundo argumento (`fallback: Date | string`) que se usa cuando el regex no matchea — normalmente se pasa `lastModified`.

**How to apply:**
- Para mostrar la hora de captura de una foto de damage, usar el pipe: `{{ url | damageImageDate: lastModified }}`.
- El util `getDamageImageName` (`core/utils/damage-image-utils.ts`) extrae solo el nombre del archivo desde la signed URL de S3 — reutilizable si necesitas el nombre limpio en otro contexto.
- Si el backend cambiara la convención de nombre, el pipe cae al fallback automáticamente; no rompe la UI.
- Priorizar la hora del filename sobre `lastModified` cuando importe la hora "real" de captura (mostrada al operador para investigar incidentes).
