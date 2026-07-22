---
name: Damage Config — estado actual en ambos proyectos
description: Feature de configuración de fotos de daño implementada en backoffice (dialog) y commerce (página de settings)
type: project
originSessionId: b4273c1a-e969-4350-954e-61a3d91558d6
---
Feature expone un parámetro: `MINIMUM_DAMAGE_PHOTOS` (INTEGER ≥ 0) por location.

**Why:** Nueva feature de producto para controlar mínimo de fotos requeridas al registrar un daño en la app de valet.

## backoffice (`feature/damage-config`, rama activa)

Implementada como **dialog** desde el dropdown "Actions" de la página `/locations`.

Archivos clave:
- `core/http/endpoints/settings/damage.ts` — endpoints
- `core/definitions/settings/damage/damage.d.ts` — tipos API
- `core/services/settings/damage/damage-config.service.ts` — GET + PATCH
- `core/states/settings/damage/damage-config.state.ts` — BehaviorSubject (no usado por dialog)
- `shared/components/damage-config-dialog/` — dialog standalone
- `pages/main/locations/location/location.component.ts` — abre el dialog

Endpoints: `GET /backoffice/v1/config-parameters/damage`, `PATCH /backoffice/v1/config-parameters/damage`
Headers requeridos: `Operator-Id`, `Location-Id`

Comportamiento del dialog:
- GET en `ngOnInit` para pre-popular el input
- `onSubmit` solo muestra snackbar de éxito + cierra — NO llama PATCH
- NO hay `afterClosed$` en el padre (damage config no afecta datos de location)

## frontend-commerce (branch no especificado, versión 3.3.0)

Implementada como **página** en `/settings/damage-config`, última opción en la sección SETTINGS del nav.

Archivos clave:
- `core/utils/path-service-setter.ts` — `DAMAGE_CONFIG.CONFIGURATION` + cutover de EV Charging y Parking Rules
- `core/definitions/damage-config.d.ts`
- `core/enums/damage-config-key.ts` — `DamageConfigKey.MinimumDamagePhotos`
- `core/states/damage-config-state.ts`
- `core/services/damage-config-service.ts`
- `pages/settings/damage-config/` — página completa
- `app.routes.ts` — ruta lazy `/settings/damage-config`
- `shared/layouts/main-layout/main-layout.component.ts` — nav item

Comportamiento de la página:
- Dropdowns de operator + location (patrón idéntico a parking-rules)
- GET al seleccionar location
- Input numérico para `MINIMUM_DAMAGE_PHOTOS`
- Sticky save bar cuando hay cambios → PATCH al guardar

## URL Cutover aplicado en commerce (2025-05-22)

Paths actualizados en `path-service-setter.ts` (los viejos ya dan 404):
- `EV_CHARGING.CONFIGURATION`: `backoffice/v1/ev-charging/configuration` → `backoffice/v1/config-parameters/ev-charging`
- `PARKING_RULES.CONFIGURATION`: `backoffice/v1/parking-rules/configuration` → `backoffice/v1/config-parameters/parking-rules`

**How to apply:** Si se agregan futuros módulos de configuración, todos viven bajo `backoffice/v1/config-parameters/{módulo}` en commerce y `backoffice/v1/config-parameters/{módulo}` en backoffice.
