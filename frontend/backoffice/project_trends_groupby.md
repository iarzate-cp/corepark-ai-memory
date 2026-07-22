---
name: Trends — Group By selector pendiente de backend
description: Feature para agregar selector de agrupamiento (Day/Week/Month) en el módulo de Trends; esperando cambio de backend
type: project
originSessionId: a82fb851-3b84-4a52-a09c-82393ffec60c
---
El módulo de Trends actualmente deja que el backend decida el agrupamiento (DAILY/WEEKLY/MONTHLY) según el rango de fechas. El request `POST /reports/trends/data` no envía `groupBy`.

**Bug reportado:** "Year to Date" agrupa por semana, debería agrupar por mes por defecto.
**Feature solicitada:** El usuario pueda seleccionar el agrupamiento para cualquier periodo.

**Plan acordado:** Opción B — agregar dropdown "Group by" en el header de Trends, con defaults sensatos por periodo.

**Solicitud enviada a backend:**
- Aceptar campo opcional `groupBy` (DAILY | WEEKLY | MONTHLY) en el request body
- Si no viene, mantener comportamiento actual (backward compatible)
- Si viene, respetarlo sobre la lógica automática
- Confirmar que `metadata.grouping` en la respuesta refleja el agrupamiento real usado

**Why:** Aún sin respuesta del equipo de backend — frontend no puede avanzar hasta que confirmen o agreguen el parámetro.
**How to apply:** Cuando backend confirme, implementar en `TrendDataRequest` (trends.d.ts) + selector en el componente + defaults por periodo.

Archivos clave:
- `src/app/pages/main/dashboard-trend/dashboard-trend.component.ts` — lógica principal, buildRequest()
- `src/app/core/definitions/trends/trends.d.ts` — TrendDataRequest, TrendMetadata
- `src/app/core/services/reports/trends/trends.service.ts` — llamada al API
