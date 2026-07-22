---
name: Variable Bulk Rate — implementación + integración pendiente
description: Feature de creación y edición bulk de Variable Rates. Componentes creados, integración en rates-buttons e index revertida por linter — pendiente re-integrar.
type: project
originSessionId: 35409f92-58a3-44bd-8ddb-af3dd22cf50e
---
## Estado actual (2026-05-13)

Los componentes y utilidades están **creados como archivos**, pero la integración en `rates-buttons.component.ts` e `index.ts` fue **revertida por el linter** en la última sesión. Necesita re-integración antes de hacer commit.

---

## Qué hace

### Crear (CreateVariableBulkRate)
Menú en Rates Actions > "Variable Bulk" abre un dialog con 4 campos:
- **Rate Name**, **Hourly Rate**, **Daily Max**, **Hours Per Day**, **Total Days**

El generador crea `totalDays × (hoursPerDay + 1)` tramos automáticamente y los manda con `rateTypeId: 3` (VARIABLE). `VARIABLE_BULK: 5` es solo un ID interno del frontend para el menú.

### Editar (auto-detección)
Al hacer click en editar una Variable Rate:
1. `inferBulkParams(rate.prices)` corre automáticamente
2. Si los tiers **siguen el patrón bulk** → abre `UpdateVariableBulkRate` pre-llenado con los 4 params inferidos
3. Si los tiers son **irregulares** → abre `UpdateVariableRate` (flujo normal, sin aviso)

**Why:** El usuario no necesita decidir nada — el sistema detecta automáticamente qué editor es apropiado.

---

## Patrón del generador (variable-bulk-tier-generator.ts)

Para cada día `d` (0 a `totalDays-1`):
- `hoursPerDay` tiers hourly: `price = d*dailyMax + h*hourlyRate`, duración = 60 min
- 1 tier cap: `price = (d+1)*dailyMax`, cubre el resto del día

Total tiers = `totalDays * (hoursPerDay + 1)`

## Lógica del inferrer (variable-bulk-params-inferrer.ts)

Inverso exacto del generador:
1. Cuenta tiers de 60 min en el día 0 → `hoursPerDay`
2. `prices.length % (hoursPerDay + 1) === 0` → si no, `null`
3. `hourlyRate = prices[0].price`
4. `dailyMax = prices[hoursPerDay].price`
5. `totalDays = prices.length / (hoursPerDay + 1)`
6. Valida el patrón completo con tolerancia `0.001` para flotantes

---

## Archivos creados (untracked — pendientes de commit)

| Archivo | Estado |
|---|---|
| `src/app/core/utils/variable-bulk-tier-generator.ts` | ✅ creado |
| `src/app/core/utils/variable-bulk-params-inferrer.ts` | ✅ creado |
| `src/app/shared/components/settings/rates/variable-bulk-rate-form/` | ✅ creado |
| `src/app/shared/components/settings/rates/create-variable-bulk-rate/` | ✅ creado (con info banner + schedule) |
| `src/app/shared/components/settings/rates/update-variable-bulk-rate/` | ✅ creado (pre-llena form con params inferidos) |

## Archivos modificados (staged — algunos revertidos por linter)

| Archivo | Estado |
|---|---|
| `src/app/core/definitions/settings/rates/rates-forms.d.ts` | ✅ OK — `VariableBulkRateFormControls` + `schedule?` |
| `src/app/core/forms/settings/rates.ts` | ✅ OK — `variableBulkRateForm()` factory |
| `src/app/core/utilities/settings/rates/rate-identifiers.ts` | ✅ OK — `VARIABLE_BULK: 5` |
| `src/app/shared/components/settings/rates/rates-actions/rates-actions.component.ts` | ✅ OK — botón Variable Bulk en el menú |
| `src/app/shared/components/settings/rates/index.ts` | ⚠️ REVERTIDO — faltan exports de `CreateVariableBulkRate`, `VariableBulkRateForm`, `UpdateVariableBulkRate` |
| `src/app/shared/components/settings/rates/rates-buttons/rates-buttons.component.ts` | ⚠️ REVERTIDO — falta lógica de auto-detección con `inferBulkParams` y `#launchEditDialog` |

---

## Re-integración pendiente

### `rates/index.ts` — agregar al final:
```ts
export { CreateVariableBulkRate } from './create-variable-bulk-rate'
export { VariableBulkRateForm } from './variable-bulk-rate-form'
export { UpdateVariableBulkRate } from './update-variable-bulk-rate'
```

### `rates-buttons.component.ts` — reemplazar `onOpenEditRateDialog` completo:

```ts
// Imports a agregar:
import { Type } from '@angular/core'             // ya existe parcialmente
import { MatDialogConfig } from '@angular/material/dialog'
import { inferBulkParams } from '@utils/variable-bulk-params-inferrer'
import { UpdateVariableBulkRate } from '../update-variable-bulk-rate'

// Método público reemplazado:
onOpenEditRateDialog(): void {
  const maxWidth = this.#dialogWidth()
  const rate = this.rate()

  if (rate.rateTypeId !== RATE_NAMES_TO_ID.VARIABLE) {
    this.#launchEditDialog(this.#dialogs[rate.rateTypeId], customMatDialogConfig(maxWidth, rate), rate.rateTypeId)
    return
  }

  const params = inferBulkParams(rate.prices)

  if (params) {
    this.#launchEditDialog(
      UpdateVariableBulkRate,
      customMatDialogConfig(maxWidth, { rate, params }),
      rate.rateTypeId
    )
    return
  }

  this.#launchEditDialog(UpdateVariableRate, customMatDialogConfig(maxWidth, rate), rate.rateTypeId)
}

// Método privado nuevo (extrae la lógica común del subscribe):
#launchEditDialog(component: Type<unknown>, config: MatDialogConfig, rateTypeId: number): void {
  this.#dialog
    .open(component, config)
    .afterClosed()
    .pipe(
      filter(Boolean),
      switchMap(() => combineLatest([this.#getRates$, this.#getSequences$]))
    )
    .subscribe({
      next: ([rates, sequences]) => {
        const message = `${this.#successMessages[rateTypeId]} rate updated successfully`
        const { action, duration, panelClass } = successSnackBarConfig()
        this.#ratesState.rates.set(rates)
        this.#ratesState.sequencesRateBean.set(sequences)
        this.#snackBar.open(message, action, { panelClass, duration })
      },
    })
}
```

---

## Decisión arquitectónica (confirmada por backend)

El backend **NO cicla** — hay que mandar los N tiers explícitamente. La implementación actual (generador que construye todos los tiers) es correcta. No hay refactor pendiente por esta razón.

**Why:** Respuesta directa del equipo de backend el 2026-05-13.

---

## UpdateVariableBulkRateData

El dialog `UpdateVariableBulkRate` recibe como `MAT_DIALOG_DATA`:
```ts
interface UpdateVariableBulkRateData {
  readonly rate: Rate       // la rate existente
  readonly params: VariableBulkParams  // inferidos automáticamente
}
```
`ngOnInit` hace `setValue` de los 4 campos del bulk form y pre-llena schedule si existía.
