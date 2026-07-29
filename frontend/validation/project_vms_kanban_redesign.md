---
name: Valet Customer Monitor — módulo kanban nuevo coexistiendo con la vista clásica
description: Kanban de 3 columnas para monitoreo de valet en /valet-customer-monitor (módulo NUEVO "Valet Customer Monitor"); la vista clásica de 4 tablas sigue viva en /valet-service-monitoring. Incluye mapping de estados, dependencias con backend y decisiones de coexistencia.
type: project
originSessionId: 3a9f09d7-195c-4a5d-bc55-de3436c6f385
---

## Contexto (2026-07-28)

**Historia**: originalmente se hizo como *rediseño* (branch `feature/vms-kanban-redesign`, reemplazaba el viejo). Luego se corrigió a *módulo nuevo* (branch `feature/coexist-vms-and-board`), y finalmente se renombró a "Valet Customer Monitor" (branch `feature/rename-valet-customer-monitor`). Los tres flows se mergearon a `develop` el mismo día, en ese orden.

Estado final:
- `/valet-service-monitoring` = **módulo clásico** (4 tablas: Pending / Retrieving / En Route / Ready). Layout original con header simple `<h1>Valet Service Monitoring</h1>` + reloj. Usa los pipes `checkIn`, `expectedCheckoutTime`, `readyToPickup`.
- `/valet-customer-monitor` = **módulo nuevo "Valet Customer Monitor"** (kanban de 3 columnas: Waiting Queue / Enroute / Parked-On Hold). Layout con brand logo Corepark (link a `/`) + título + station filter inline en el header oscuro. Usa `<valet-monitoring-board />` shared component (el shared component conserva su nombre porque es UI reutilizable, no es la identidad del módulo).

Los flows se mergearon a `develop` (no a `main`) por decisión explícita del usuario.

## Coexistencia — cómo se resuelve el estado compartido

`ValetServiceMonitoringState` es la fuente única de datos crudos (Firebase + station filter). Expone **dos sets de computeds derivadas** para no romper ninguno de los dos módulos:

- Set clásico: `pendingTickets`, `attendTickets`, `enRouteTickets`, `readyForPickUpTickets` — lógica original con `RequestNotificationStatus.Pickup` como discriminador
- Set nuevo (prefijo `board*`): `boardWaitingQueueTickets`, `boardEnRouteTickets`, `boardParkedOrOnHoldTickets` — todas ordenadas por `time` ascendente

**Why:** las computeds del set nuevo tienen semántica distinta (por ejemplo, `boardEnRouteTickets` incluye TODOS los EN ROUTE mientras que `enRouteTickets` clásico excluye los con notification `Pickup`). Si comparten nombre habría colisión y regresiones sutiles en ambos módulos. Prefijar el set nuevo con `board*` es más seguro que renombrar el clásico (que ya estaba en producción).

**How to apply:** al agregar UI en el módulo nuevo, siempre usar `state.board*`. Al tocar el módulo clásico, usar los nombres sin prefijo. Nunca mezclar.

## Mapping de estados a columnas en el kanban (`board*`)

Decisión tomada tras investigar la app Android `mobile_worker`. Ver `com.corepark.valet.newarch.request_manager_journey.domain.model.AttendingTicketStatus`.

- **Waiting Queue** = `status = REQUEST` (mostrado como "Awaiting an attendant") o `status = ATTEND` (mostrado como "Accepted · &lt;name&gt;"). Contraintuitivo: **ATTEND no va en Enroute**, va en Waiting Queue. Semánticamente ATTEND es "valet aceptó pero aún no escanea llaves" → sigue siendo espera desde el punto de vista del guest.
- **Enroute** = `status = EN ROUTE` solamente. Copy dice "Keys scanned — attendant is driving from one garage to another".
- **Parked / On Hold** = `status = PARK` (badge "PARKED ROW &lt;x&gt;") o `status = HOLD` (badge "HOLD &lt;x&gt;").

**Why:** el copy de cada columna en el diseño describe una acción específica del ciclo de vida. "Keys scanned" es exclusivo de EN ROUTE. ATTEND ocurre antes de escanear llaves.

## Orden dentro de cada columna del kanban

Todas las columnas se ordenan por `ticket.time` **ascendente** (primer ticket = próximo a atender). `time` es el timestamp de la última transición de estado, no `checkIn` (que es cuándo llegó el carro).

**Why:** producto quiere que la primera fila siempre sea el próximo carro a entregar. En Waiting Queue = el que lleva más esperando; en Enroute = el que empezó a moverse primero (más cerca de arribar); en Parked = el parqueado hace más tiempo.

## Campos que llegan gradualmente en el ticket

La respuesta de Firebase **no incluye todos los campos siempre** — dependen del estado. La interface `Ticket` los marca como opcionales para forzar acceso defensivo:

- `maker`, `model`, `color`, `plate`, `phoneCodeId`, `phoneNumber`, `employee` — no vienen en `CHECK-IN`, sí en `ATTEND` en adelante
- `enRoute { origin { parkingAreaId, parkingAreaName }, destination { ... } }` — solo cuando el atendedor marcó destino. Nodo **sticky**: persiste en PARK/HOLD/COMPLETE, hay que filtrar por `status === 'EN ROUTE'` antes de mostrarlo
- `spotAssignment { area { id, name }, row, spot }` (nuevo) o `row`/`spot` top-level (legacy) — para el badge de la col 3
- `requestNotification.status = ACCEPTED` — **pendiente que backend popule**. Sin esto, no hay forma de mostrar "Accepted · &lt;name&gt;" en la col 1 aunque diseño lo pida; se usa `status === ATTEND` como proxy.

**How to apply:** cualquier lectura de estos campos debe usar `?.` + `??` con fallback (ver `feedback_optional_chaining`). El template ya lo hace: `maker ?? '—'`, y `parkingArea(ticket)` retorna `''` para que el `@if` esconda "at &lt;location&gt;".

## Station filter — dos comportamientos visuales

El componente vive en `shared/components/station-filter/`. Por defecto tiene chrome (border, padding) — así lo usa el módulo clásico. El módulo nuevo strippea la chrome via CSS custom properties (`--station-filter-border`, `--station-filter-padding`, etc.) desde su layout. Los reset a `transparent` / `0` para que en el header oscuro se vea como texto.

Además: si solo hay 1 station activa, renderiza como texto plano sin dropdown (regla de UX aplicable en ambos módulos). Ver `project_monitoring_station_filter.md` para las reglas producto originales.

## Ambiente / verificación

Al mergear a develop, solo un ticket real fue validado visualmente (ticket #6 en ATTEND, KIA, Israel Arzate → aparece en Waiting Queue con "Accepted · Israel A."). **Enroute y Parked/On Hold del módulo nuevo no fueron verificadas con tickets reales** porque no había tickets en esos estados en Firebase al momento del merge. Reviewer debería validar en un ambiente con actividad real.

## Cosas intencionalmente NO hechas

- **Íconos de marcas de vehículos**: se muestra solo el texto de `maker`. svgrepo tiene los logos bajo "Logo License" (permite uso identificativo con restricciones), pero descarga manual y muchas marcas — se pospuso.
- **i18n con ngx-translate**: proyecto es US-only, todo el copy vive hardcoded en inglés en el template del board.
- **Tests unitarios de los utils** (`employee-name-formatter`, `ordinal-formatter`, `spot-formatter`): saltados por prisas del release. CLAUDE.md pide tests para utils, agregar si se retoma.

## Ícono `corepark-white-icon`

Refactorizado para exponer `size` y `fill` como signal inputs (defaults `2rem` / `#fff`). El sizing lo controla `@HostBinding` en el host, no styles externos. El módulo nuevo lo usa con `size="2.75rem"`.
