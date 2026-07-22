---
name: Stations domain — endpoint, invariantes y naming en el BO
description: Dónde vive el dominio de "stations" (valet), qué garantiza el backend, y por qué en frontend-backoffice está bajo el nombre "transfer-ticket"
type: reference
originSessionId: 86ddfc26-ea4f-451b-90ba-82004ad47d51
---
## Endpoint

- `GET /backoffice/v1/station` — devuelve `{ stations: Station[] }`
- Headers requeridos: `Operator-Id`, `Location-Id`
- Base URL: misma que el resto de servicios (`environment.endpoint`)

**Shape de `Station`:**
```ts
{ uuid, name, createdAt, deactivatedAt: string | null, default: boolean }
```
`StationOverview` extiende con `ticketSequenceCount: number` (endpoint `/station/overview`).

## Invariantes de backend (no codificadas en el frontend)

- **Toda location tiene exactamente una station con `default: true`.**
  Esto justifica que UI de filtros siempre pueda pre-seleccionar la default sin plan B para "sin default".
- `deactivatedAt !== null` → station desactivada; no aparece en selectores de filtro (pero sí sigue en la lista raw).

## Naming quirk en frontend-backoffice

En `frontend-backoffice`, todo el dominio de stations vive bajo el prefijo `transfer-ticket-*`, no `station-*`:

- Ruta pública: `/settings/stations` (`app.routes.ts`)
- Página: `TransferTicketPageComponent` (`pages/main/transfer-ticket-page/`)
- Servicio: `TransferTicketService` (`core/services/transfer-ticket-service.ts`) — **es el que expone el CRUD real de stations + ticket-sequences**
- State: `TransferTicketState` (`core/states/transfer-ticket-state.ts`)
- Definiciones: `core/definitions/transfer-ticket.d.ts`
- Componentes: `shared/components/transfer-ticket-{add,card,edit,deactive,...}/`
- Códigos de error: `STATION-003` (no borrar default), `STATION-006` (no marcar default a desactivada)

Cuando busques "stations" en el BO por primera vez, la mayoría de hits van a ser "Windcave station" (dominio de pagos), no valet. Filtra por `transfer-ticket` o busca `interface Station` para encontrar el dominio correcto.

## Relación con Firebase (monitoring)

Los tickets emitidos por Firebase incluyen `assignedStationUuid?: string` que vincula el ticket a una station por UUID. La interfaz `Ticket` de `frontend-validation` no lo declaraba históricamente — el payload real trae más campos de los que la interfaz reflejaba.
