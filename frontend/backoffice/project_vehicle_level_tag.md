---
name: Vehicle Level Tag — PM-2648
description: Feature de tags por vehículo (ALL_VEHICLES / SELECTED_VEHICLE) en Guest Profiles — COMPLETA y mergeada a develop en v1.23.0
type: project
originSessionId: 376a91b8-fb58-42a0-be23-e2e4e071f72b
---
Feature convierte `tag: string | null` en `tags: TagDto[]` por perfil y por car. Cada tag tiene scope: ALL_VEHICLES (aplica a todos los cars del perfil) o SELECTED_VEHICLE (aplica a un car específico, 1:1).

**Why:** Clientes como hoteles y country clubs usan tarjeta de membresía global; Tesla Assembly Plant usa sticker por vehículo. El modelo anterior (string único) no cubría la coexistencia.

**Estado actual:** Mergeada a develop en v1.23.0. Branch `feature/vehicle-level-tag` en commit `74cde031`.

---

## Archivos modificados / creados

### Core
- `core/enums/guest-settings/profiling.ts` — `TagType` enum: `AllVehicles = 'ALL_VEHICLES'`, `SelectedVehicle = 'SELECTED_VEHICLE'`
- `core/definitions/guest-settings/profiling.d.ts` — `GuestProfile.tags: TagDto[]`, `Car.tags: TagDto[]`, interfaces `TagDto`, `TagsResponse`, `CreateTagPayload`, `UpdateTagPayload`, `SearchBy`, `SearchGuestProfileDto`, `SearchGuestProfilesResponse`, `SearchGuestProfilesPayload`; removido `tag: string | null`
- `core/services/guest-settings/profiling/profiling.service.ts` — endpoints: `getTags`, `createTag`, `updateTag`, `deleteTag`, `searchGuestProfiles`; `#locationHeaders` computed; `mapSearchDtoToGuestProfile` pure fn (mapea `guestId→id`, `carProfiles→cars`)
- `core/states/guest-settings/profiling/profiling.state.ts` — signal `tags: TagDto[]`; `clear()` la resetea

### Shared components
- `guest-profile-edit/` — removido campo `tag` del form, validadores y payload
- `guest-profile-tag-dialog/` — componente nuevo (create/edit modal):
  - Branch A: scope ALL_VEHICLES por default
  - Branch B: scope SELECTED_VEHICLE con car pre-seleccionado (`preselectedCarId`)
  - `>5 cars` → `mat-select` en lugar de radios
  - Variante A: radio "Selected vehicle" disabled con "Add a vehicle first" si no hay cars
  - Variante B: validación `^.+\d$`, error inline, Save disabled
  - Variante C: preview "Will also apply to: ..." al cambiar Selected→Global en edición
  - `ticket` derivado automáticamente con `/(\d+)$/` del scanCode
  - Cierra con `TagDto[]` (response del endpoint); padre actualiza inline sin re-fetch
  - **`TagDialogData` incluye `cars: Car[]` — el padre los pasa, el dialog NO los fetchea**
- `guest-profiling-table/` — columna TAG con chips ámbar + X (delete) + "+ Add tag" dashed; botón `actioner-raised color="main"` con `mat-menu` para search by (All fields / Tag / First name / Last name / Phone / Plate / VIN); búsqueda TAG server-side, resto client-side; `searchBy` signal + `searchByLabel` computed; `#tagSearchResults` signal override del state para resultados de búsqueda
- `guest-profile-cars/` — columna TAG con chips ámbar ALL_VEHICLES (click edita) + chip azul SELECTED_VEHICLE con X + dashed "+" si no tiene específico; leyenda "⊕ Global ⊟ Específico" alineada a la derecha (texto en español — pendiente traducir a "Specific"); paginator conectado via `effect()`; `onOpenGlobalTagDialog`, `onDeleteCarTag`, `onOpenTagDialog`, `#refreshCarTags`
- `guest-profile-delete-car/` — nueva interfaz `DeleteCarDialogData { car, selectedVehicleTag? }`; warning en rojo si el car tiene SELECTED_VEHICLE tag (Variante D cascade)

---

## Patrones clave
- `ticket` derivado en frontend con `/(\d+)$/` — el backend no lo deriva, pero lo espera
- Chips: ámbar = ALL_VEHICLES (editable desde perfil y car), azul = SELECTED_VEHICLE (editable + X desde car), dashed = agregar
- Estado siempre actualizado inline tras CRUD de tags (sin re-fetch del perfil)
- Search by TAG usa `POST /backoffice/guest-profiles/search`; response tiene shape diferente (`guestId`, `carProfiles`) mapeado via `mapSearchDtoToGuestProfile`
- `DeleteCarDialogData` reemplaza el `Car` directo como dato del dialog de delete car
- El caller pasa `cars` al dialog via `TagDialogData` — no intentar mover el fetch adentro del dialog

## Issue visual conocido
El campo "Scan value" muestra borde naranja/rojo al abrir el dialog porque `cdkFocusInitial` lo enfoca inmediatamente y el control ya está inválido (vacío falla `endsWithDigit`). Angular Material activa el error state al marcar el control como `touched`. Se exploró un `DirtyErrorStateMatcher` pero se descartó — no era necesario según el criterio del equipo.

## Pendiente (fuera de scope)
- **"Específico" → "Specific"** en la leyenda de `guest-profile-cars` — se hizo pero se revirtió con el rollback de sesión; pendiente como commit limpio
- **Bulk import xlsx** — columnas "Scan Code" (col 3) y "Tag Type" (col 5, default ALL_VEHICLES); no existe componente ni endpoint de importación
- **Auto-expand post-creación** — perfil nuevo debe aparecer expandido con fondo `#FFF8F0`; no existe dialog de "New profile" en el codebase
