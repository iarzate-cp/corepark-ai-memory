---
name: Ticket Log — columna Room Number (backoffice + microservice-reports)
description: Columna Room en el Ticket Log del BackOffice. Rama feature/ticket-log-room-number en AMBOS repos (desde main), commiteada, pusheada y mergeada a feature/staging. Backend expone hotelRoom en get-ticket-list; front lo mapea a room. Faltan las PRs a main.
type: project
---

## Alcance del ticket

"Add Room Number field to BackOffice Ticket Log module" — que el operador vea el room en la tabla sin abrir cada ticket. Interpretado como el **Ticket Log del BackOffice** (reporte genérico operador-facing), NO el Activity Log per-ticket de la Valet App. Distinto de TCK-48 (waiving room requirement en checkout) y TCK-67 (bug de timestamp en valet app).

## Estado — cerrado 2026-08-13

| Repo | Feature branch (base `main`) | Merge a `feature/staging` | Push |
|---|---|---|---|
| `frontend-backoffice` | `5acd67b7` | `892a4480` | ✓ ambas |
| `microservice-reports` | `88f1e91` | `cea78ea` + `c68f478` | ✓ ambas |

**Pendiente:** las PRs a `main` en ambos repos. Nadie las ha abierto.

**Trampa que costó un paso extra:** el backend se editó estando parado en `feature/staging`, que va **198 commits adelante de main**. Hubo que mover los cambios con `stash → checkout main → branch → pop`. Aplicó limpio porque `TicketBasicInformationDto.java` y `TicketListReportDaoImpl.java` son idénticos entre main y staging; el único que difiere es `TicketListReportServiceImplTest.java` (staging tiene `ReceiptService` / feature `ticketReceipt`, main no). El `pop` auto-mergeó y dejó la versión de main con el edit encima — que es lo correcto para una rama que apunta a main.

**Lección:** antes de editar backend, verificar en qué rama se está parado. El default de la working copy es `feature/staging`, no `main`.

## Las dos divergencias del merge a staging (2026-08-13)

Los dos repos estaban desfasados, en direcciones opuestas. Ninguna dio conflicto, pero ambas cambian lo que ves en el diff del merge.

**Backend — staging local 10 commits ATRÁS de origin.** Mergeé la feature sobre la base vieja antes de que el `fetch` lo revelara; se arregló con `git merge origin/feature/staging` antes de pushear (patrón de [[feature_hotel_info_gate]]). Lo que faltaba: el **External Data API v1** completo (3 merges) y `fix(ticket-list): populate rateChangeComment in the Excel export` — este último toca el MISMO módulo, por eso se corrió la suite completa (224 tests) y no solo la clase del ticket-list. Verde.

**Front — staging desfasado de main en la otra dirección.** El merge le metió de paso 3 commits que ya estaban en main y staging no tenía (`chore/youtube-videos` + el video de training). Por eso aparece `training-videos-page.videos.ts` en el diff del merge sin que la feature lo tocara.

`feature/staging` es la rama de pruebas del equipo en ambos repos. **Siempre `git fetch` + `git rev-list --left-right --count` antes de mergear ahí.**

## Backend — 3 archivos

- `TicketListReportDaoImpl.java` — `QRY_TICKET_LIST` ahora selecciona `TRIM(ps.hotel_room) hotelRoom`. **TRIM porque la columna trae padding**: `InventoryReportDaoImpl` ya la envuelve en `COALESCE(TRIM(...))`, y hay precedente de trailing spaces en esta tabla (ver `logRate` en [[project_activity_log_status_rendering]]). `TRIM(NULL)` sigue siendo `NULL`.
- `TicketBasicInformationDto.java` — campo `hotelRoom` **agregado al final**. El DTO usa `@AllArgsConstructor` posicional y el DAO lo construye posicionalmente → agregar un campo cambia la aridad y rompe `TicketListReportServiceImplTest.createMockTickets()`. Al final = un solo arg extra por línea, mínimo churn.
- `TicketListReportServiceImplTest.java` — 3 mocks actualizados.

**El export a Excel/CSV NO se tocó**: ya traía Hotel Room desde antes (`TicketListReportServiceImpl.java:338` xlsx y `:428` csv, header `"Hotel Room"` asertado en el test del CSV).

## Frontend — 8 archivos

- `ticket-log.d.ts` — `Ticket.hotelRoom: string | null`, `TicketLogDataSource.room`. **Gotcha:** `TicketDetailParams = Omit<Ticket, ...>` — al agregar un campo a `Ticket` hay que sumarlo al `Omit` o `openDialog()` en el table deja de compilar.
- `ticketLogFactory` — mapea `hotelRoom → room`.
- `TicketLogLayoutState` — columna `room` después de `phone` (agrupa con datos del huésped, mismo criterio que Overnight: guest → room). Móvil sigue en `['ticket','actions']`.
- Tabla + `ticket-log-detail` — celda con fallback `-`; fila "Room number" en el bloque Guest Information del diálogo, con `room()` metido en `hasGuestInformation()` para que un ticket con solo habitación y sin nombre igual muestre el bloque.
- `en.json` / `es.json` — `TICKET_LOG.TABLE.ROOM` y `TICKET_LOG_DETAIL.GUEST_INFORMATION.ROOM`. `id.json` no tiene el namespace `TICKET_LOG`, no se tocó.

**Bonus gratis:** al meter `room` en `TicketLogDataSource`, el buscador de la tabla (`MatTableDataSource.filter`) lo indexa solo — se puede buscar por habitación sin código extra.

**Hallazgo:** el diálogo de detalle del BackOffice **nunca mostró el room** aunque el servicio ya mapeaba `hotelInformation`. La premisa del ticket ("sin abrir cada ticket") era falsa: no se veía ni abriéndolo. Se agregó en el mismo cambio.

## Nomenclatura — hay tres nombres para lo mismo

`company.parking_service.hotel_room` se expone como:

| Contexto | Key |
|---|---|
| Firebase RT DB (lee mobile_worker) | `hotelRoom` |
| REST valet / guest-page (`Guest`) | `room` |
| reports `get-ticket-detail` | `hotelInformation.room` |
| reports `get-ticket-list` (**nuevo**) | `hotelRoom` |

Se eligió `hotelRoom` en la lista por consistencia con `TicketList.Ticket.hotelRoom` del mismo servicio (el DTO del export). El front lo renombra a `room` en el factory. Ver [[feature_hotel_info_gate]] para el origen de la doble nomenclatura.

## Realidad del dato — quién llena hotel_room

No es un campo que llene el BackOffice. Lo escribe **ms-valet-service vía integración PMS/Opera** en check-in, y desde la feature Hotel Info Gate también el guest page (opt-in por operador vía `guest_page_cfg.allow_hotel_room`, DEFAULT FALSE).

Fill rate real (per DDL_2026_08_04_HOTEL_INFO_GATE, datos de prod ~30 días): **19.2% de los tickets** tienen `hotel_room`, 876K filas históricas. O sea ~1 de cada 5 filas mostrará valor y el resto `-`; en locations de hotel la proporción es mucho más alta. No es una columna vacía, pero tampoco universal — si alguien reporta "la columna sale vacía", lo más probable es que sea una location no-hotel, no un bug.

## Orden de deploy

El front pinta `-` hasta que `microservice-reports` esté arriba con el campo. No rompe nada, pero si el front sale primero los operadores ven una columna Room vacía en todos los tickets. Backend primero.

## Verificación

- Backend sobre `feature/staging` ya integrado: **224 tests, 0 failures, 3 skipped, BUILD SUCCESS** (suite completa, incluye los contract/parity tests del External API recién mergeado).
- Front sobre `feature/staging` ya integrado: `ng build --configuration=dev` limpio, sin errores ni warnings.
- Ver [[reference_maven_local_setup]] — correr los tests del backend en local tiene dos trampas (wrapper viejo + modo offline).

## Deuda saldada de paso

`Activity.logRow`, `logSpot`, `room`, `expectedDeparture` en `ticket-log.d.ts` estaban como `any`; [[project_activity_log_status_rendering]] pedía tightenearlos al tocar el archivo. Hecho: los cuatro a `string | null`, más `HotelInformation.room` / `.expectedDeparture`. Build pasa.
