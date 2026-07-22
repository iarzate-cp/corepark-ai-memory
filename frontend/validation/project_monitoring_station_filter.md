---
name: Valet Service Monitoring — filtro por station (feature/filter-monitoring-by-station)
description: Decisiones de diseño no obvias del filtro por station en la página de monitoring
type: project
originSessionId: 86ddfc26-ea4f-451b-90ba-82004ad47d51
---
## Contexto

Branch: `feature/filter-monitoring-by-station`. Se añadió un `mat-menu` en la página `/valet-service-monitoring` para filtrar los 4 buckets (Pending, Retrieving, En Route, Ready) por la station asignada al ticket (`ticket.assignedStationUuid`).

## Decisiones de diseño

- **No hay opción "All stations".** La UX exige que siempre haya una station seleccionada. Justificación: toda location tiene una station con `default: true` (invariante backend), así que el estado "sin filtro" no aplica al modelo de negocio.
  - **Why:** producto quiere que operadores se enfoquen en su station; ver todo mezclado fue la causa raíz de esta feature.
  - **How to apply:** si alguien pide "mostrar todos", replantear — probablemente quieran una view distinta, no romper este contrato.

- **Mientras carga la lista de stations, se muestra vacío (no todos los tickets).** El computed base `tickets` en `ValetServiceMonitoringState` devuelve `[]` si `selectedStationUuid` es `null`.
  - **Why:** evita un "flash" de tickets que después desaparecen al llegar la default. Consistente con "siempre filtrado por default".
  - **How to apply:** si en el futuro se ve "no tickets" al entrar y hay reporte de bug, verificar primero si `StationsService.getStations()` respondió y si `defaultStation` computed devolvió algo — no es un bug del filtro de tickets, es que la lista de stations no llegó.

- **Fallback si no hay `default: true`**: se selecciona la primera station activa (`activeStations().at(0)`). El contrato "siempre filtrado por una station" se mantiene aunque el invariante backend se rompa.
  - **Why:** producto prefirió no bloquear al operador con la vista vacía si backend olvida marcar default. Se descartó "mostrar todos los tickets" porque rompía el foco por station (motivo original de la feature).
  - **How to apply:** si en algún momento se pide un modo "sin filtro", replantear — probablemente sea una view distinta, no ampliar este estado. Si al operador le aparece una station "inesperada" como seleccionada, revisar en backend que la location tenga su `default: true`.

- **Solo se listan stations activas en el menú** (`deactivatedAt === null`), pero el filtro por UUID funciona incluso si la station fue desactivada después de asignarse a un ticket vivo.

## Archivos clave

- Filtro: `shared/components/station-filter/` (mat-menu + trigger)
- Data: `core/services/stations-service.ts` + `core/states/stations-state.ts`
- Lógica: `ValetServiceMonitoringState.selectedStationUuid` + computed `tickets`
- Página: `pages/valet-service-monitoring-page/` (carga stations en `ngOnInit`, setea default UUID)
