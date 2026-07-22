---
name: Garage Spot Uniqueness Toggle — enforceSpotSingleOccupancy
description: Feature que agrega toggle por garage para rechazar park/relocate en celda ya ocupada
type: project
originSessionId: 28fcfc49-958f-4478-af5a-24c5693e47e1
---
Feature implementada en `feature/garage-spot-uniqueness-toggle` (backoffice). Pendiente de commit y PR.

**Why:** Producto pidió que cada garage pueda opt-in a una regla que rechaza park/relocate cuando la celda destino (row + spot) ya está ocupada por otro ticket activo. El enforcement ocurre en ms-valet-service; backoffice solo persiste el flag.

**How to apply:** Si hay cambios futuros al formulario de garage (nuevos toggles, nuevas reglas por garage), van al mismo bean `ParkingArea` sin endpoints nuevos — el spec lo menciona explícitamente como patrón a seguir.

## Archivos modificados

- `core/definitions/spot-config-service.d.ts` — `enforceSpotSingleOccupancy: boolean` en `ParkingLayoutArea`
- `core/definitions/spot-config-forms.d.ts` — `enforceSpotSingleOccupancy: FormControl<boolean>` en `ParkingLayoutFormControls`
- `core/services/spot-config-service.ts` — nuevo param en `createParkingLayoutArea` y `updateParkingLayoutArea`
- `spot-config-toggle-collapse.form.ts` — `new FormControl<boolean>(false)` en el form group
- `spot-config-toggle-collapse.component.ts` — set en `#setForm`, reset en `enableEdit`/`onCancel`, enviado en `#request`
- `spot-config-toggle-collapse.component.html` — `<mat-slide-toggle color="primary">`
- `spot-config-new-garage.component.ts` — destructura y pasa el flag en `#request`
- `spot-config-new-garage.component.html` — `<mat-slide-toggle color="primary">`
- Ambos `.imports.ts` — `MatSlideToggleModule`

## Comportamiento clave

- El PUT siempre debe incluir el flag actual (no hay endpoint dedicado para el toggle aislado)
- `onCancel()` resetea el valor del toggle al original del `parkingLayoutArea` para no mostrar valor sucio en modo read
- Default: `false` en garages existentes (columna BD es NOT NULL DEFAULT FALSE)
