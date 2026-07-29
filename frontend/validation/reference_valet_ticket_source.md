---
name: Valet ticket source — mobile_worker es quien escribe a Firebase
description: Los tickets de valet los escribe la app Android mobile_worker directamente a Firebase RTDB. Este repo (y el resto del stack web) solo LEE. Fundamental para entender por qué ciertos campos llegan gradualmente y dónde investigar cuando la data no viene.
type: reference
originSessionId: 3a9f09d7-195c-4a5d-bc55-de3436c6f385
---
## Arquitectura

**Escritor**: app Android `mobile_worker` (`/Users/israel/Dev/mobile_worker`, GitHub `corepark/mobile_worker`, package `com.corepark.valet`).
**Storage**: Firebase Realtime Database.
**Path**: `/parkingServices/operatorCompany/{operatorCompanyId}/parkingLot/{parkingLocationId}/ticket/{ticketId}`
**Lectores web**: frontend-validation (VMS + Valet Customer Monitor), frontend-valet-web, entre otros.

Este repo **no puede modificar el shape de los tickets** — cualquier campo faltante o mal formateado se resuelve en mobile_worker + backend (`ms-valet-service`).

## Campos que llegan gradualmente

El objeto en Firebase se enriquece según avanza el ciclo de vida del ticket:

- **CHECK-IN**: shape mínimo (ticket, checkIn, employee, ixParkingLocation*, parkingLocationId, operatorCompanyId, rate, rateVersion, status, time). NO trae maker/model/color/plate/plate/phone*.
- **ATTEND en adelante**: se suman datos del vehículo (`maker`, `model`, `color`, `plate`), del huésped (`firstName`, `lastName`, `phoneCodeId`, `phoneNumber`), y `expectedCheckoutTime`.
- **EN ROUTE**: nodo `enRoute { origin, destination }` con `parkingAreaId` + `parkingAreaName`. **Sticky**: persiste en PARK/HOLD/COMPLETE, filtrar por `status === 'EN ROUTE'` antes de mostrarlo.
- **PARK / HOLD**: `spotAssignment { area { id, name }, row, spot }` (nuevo) o `row`/`spot` top-level (legacy).

## Dónde investigar en mobile_worker cuando algo falta

- **Data models / DTOs**: `app/src/main/java/com/corepark/valet/ui/models/ticket/` (ver `TicketModel.kt`, `RequestNotification.kt`, `EnRouteNode.kt`)
- **Transiciones de estado**: `app/src/main/java/com/corepark/valet/newarch/check_out_journey/ticketlist/ChangeTicketStatusUseCase.kt`, `app/src/main/java/com/corepark/valet/newarch/request_manager_journey/domain/usecase/AcceptPendingRequestUseCase.kt`
- **Enum de estados**: `app/src/main/java/com/corepark/valet/ui/models/ticket/TicketStatus.kt` + `AttendingTicketStatus.kt`
- **Formato de `employee`**: siempre `"${firstName} ${lastName}".trim()` (ver `AcceptPendingRequestUseCase.kt`)

## `requestNotification` — a quién le toca

El Android envía `AttendRequestCarDto { employeeName, employeeId, ... }` al backend cuando el valet acepta. Es el **backend (`ms-valet-service`)** quien debería actualizar `requestNotification.status = ACCEPTED` en Firebase. mobile_worker NO lo escribe directamente. Si `requestNotification.status = ACCEPTED` no aparece en Firebase, el bug es en backend, no en mobile_worker ni en el frontend.

## Cross-referencia

Ver también `project_vms_kanban_redesign.md` para el mapping detallado de estados a UI (Waiting Queue / Enroute / Parked-On Hold) en el módulo Valet Customer Monitor.
