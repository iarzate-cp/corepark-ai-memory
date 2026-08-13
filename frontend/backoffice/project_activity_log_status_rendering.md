---
name: Activity Log — reglas de render contextual por status
description: Reglas non-obvias para renderizar detalles por evento del activityLog (From/To en RELOCATE, Room en EDIT HOTEL INFO, Rate en RATE CHANGE); patrón de pipes
type: project
---

En `TicketLogDetail.activityLog` cada evento trae campos genéricos (`logRow`, `logSpot`, `logRate`, `room`, `expectedDeparture`, etc.), pero el significado depende del `status`. La UI oficial (app móvil) renderiza detalles contextuales por debajo del título/empleado de cada evento, y algunos requieren lógica más allá del propio evento.

**Why:** El backend no envía "From/To" pre-computado en RELOCATE — solo trae el destino. La UI móvil lo deriva localmente caminando hacia atrás en el activityLog. Sin esa regla, uno mira los datos del evento y no entiende de dónde sale el "From". Se replicó ese comportamiento en backoffice para el `TicketLogDetail` (rama `adjust/activity-log-full-data`, mergeada a `feature/staging` como `ea1e4970`).

**How to apply:**

1. **RELOCATE — From/To**
   - `To` = `event.logRow ?? event.logSpot` del evento actual (uno u otro, nunca ambos a la vez en la práctica).
   - `From` = último `logRow ?? logSpot` no-null caminando hacia atrás desde el índice actual en `activityLog`. Si nada previo tiene ubicación, no se muestra "From:".
   - `logRow` (fila, ej. `"Z"`) y `logSpot` (cajón, ej. `"4"`) son semánticamente distintos pero se unifican como "ubicación" a efectos de este render.
   - Implementado en `core/pipes/relocate-location-pipe.ts`. Firma: `activityLog | relocateLocation:$index` porque necesita historial + índice.

2. **EDIT HOTEL INFO — Room / Expected Departure**
   - Ambos vienen en el evento mismo (`activity.room`, `activity.expectedDeparture`). Se muestra cada uno solo si no es null.
   - Implementado en `core/pipes/activity-hotel-info-pipe.ts`. Firma: `activity | activityHotelInfo` → `{ room, expectedDeparture } | null`.

3. **RATE CHANGE — Rate**
   - Solo muestra el destino (`activity.logRate`), no from/to (aunque CHECK-IN también trae `logRate` y podría deducirse un origen — la app oficial no lo hace).
   - **Ojo con trailing space**: el backend envía `logRate: "Valet "` con espacio al final en CHECK-IN. El pipe hace `.trim()`.
   - Implementado en `core/pipes/activity-rate-pipe.ts`.

4. **Patrón general — un pipe por dato contextual**
   - Israel prefiere pipes (Angular best practice) sobre lógica inline en template o métodos de componente para estos renders condicionales por status.
   - Convención de naming: `activityXxx` para pipes que toman `Activity`; `relocateLocation` (o similar) cuando necesita el arreglo completo + índice.
   - Se registran en `activity-log.imports.ts`, se consumen con `@let x = ... | pipe; @if (x) { ... }` en el template.
   - Los detalles se envuelven en `<span class="activity-detail">` (font-size `--fs-medium`, más chico que el resto del span default).

5. **Statuses del enum que se añadieron en esta rama** (`ticket-log.enums.ts`): `EnRoute = 'EN ROUTE'`, `Hold = 'HOLD'`, `Transfer = 'TRANSFER'`. Los tres iconos SVG están en `activity-log-icons.component.html` (Tabler: `car-suv`, `clock-pause`, `transfer` respectivamente).

6. **Bug lateral encontrado** — `FullPhonePipe` (`core/pipes/full-phone/full-phone.pipe.ts`) llamaba `value.toString()` antes del guard, crasheando cuando `guestPhoneNumber` viene `null`. Se corrigió en el mismo push (commit `1ece78e6`). Considerar el mismo patrón (guard primero) en otros pipes similares.

**Tipado — deuda SALDADA:** en `core/definitions/reports/ticket-log/ticket-log.d.ts`, la interface `Activity` tenía `logRow`, `logSpot`, `room`, `expectedDeparture` como `any`. Se apretaron a `string | null` (junto con `HotelInformation`) en la rama `feature/ticket-log-room-number` — ver [[project_ticket_log_room_number]].
