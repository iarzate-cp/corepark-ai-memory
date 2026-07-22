---
name: Variable rate — estado actual, UX propuesto y refactor pendiente
description: Análisis completo del flujo de variable rate: escala real (50+ filas), 8 friction points, rediseño UX con generador de tramos y virtual scroll, deuda técnica pendiente.
type: project
originSessionId: a842525e-7ac3-42be-a7fa-5b793a2819fa
---
El refactor completo del variable rate **no fue implementado**. Solo se mergeó el parche de bug (PR #44).

## Escala de uso real

Las tarifas variables pueden tener **50+ tramos**. Este es el caso de estacionamientos de larga estancia donde vehículos permanecen días. Esta escala convierte problemas menores en bloqueantes reales.

## Estado actual (verificado 2026-05-13 — refactor implementado)

| Ítem | Estado |
|---|---|
| Bug `#parkedTimeToNumber` sin colon | Resuelto — PR #44 merged |
| `price-time-input` component | ✅ Implementado — `shared/components/settings/rates/price-time-input/` |
| `parked-time-utils.ts` | ✅ Implementado — `core/utils/parked-time-utils.ts` + tests |
| `PriceFormControls` con `FormControl<number\|null>` | ✅ Implementado |
| `variable-rate-form` standalone | ✅ Implementado |
| `variable-time` (picker flotante) | ✅ Eliminado |
| Generador de tramos bulk | ✅ Implementado en `create-variable-rate` |
| CDK Virtual Scroll | ⏳ Pendiente — deferred para siguiente iteración |
| `validation-form` migrada a `price-time-input` | ✅ Implementado |

## 8 Puntos de fricción UX (verificados en código)

### Escala pequeña
1. Picker flotante interrumpe el flujo — aparece en focus, compite con escritura directa.
2. "From" no se sincroniza cuando el picker usa `setValue` — `ngModelChange` solo se dispara en interacción de usuario, no en programático.
3. Último tramo siempre "Max" en gris — el usuario no sabe hasta cuándo cubre.
4. Sin resumen de cobertura total.
5. Botón X deshabilitado si el form es inválido (`!form.valid` en condición del template).

### Críticos al escalar a 50+ tramos
6. **"Add row" bloqueado mientras form inválido** — `[disabled]="!form.valid"` en `create-variable-rate.component.html:13` y `update-variable-rate.component.html:15`. Para 50 filas = 50 ciclos obligatorios fill → add.
7. **"Add row" en el header del dialog** — no inline al pie de la lista. Scroll friction constante.
8. **Sin bulk/generador** — `addPriceGroup()` agrega 1 fila vacía a la vez. 50 tramos = 50 interacciones manuales independientes.

## Rediseño UX propuesto (diseñado 2026-05-13)

### Cambio central
Reemplazar modelo "from/to por fila" por "segmentos con corte":
- `From` → etiqueta estática auto-derivada (no input)
- `To` del último → `∞` visible (no "Max" disabled confuso)
- "Add row" → al pie del FormArray, siempre habilitado

### Generador de tramos (para 50+ filas)
Paso de configuración antes del FormArray:
- Intervalo de tiempo (ej. 1h)
- Duración total (ej. 72:00 = 3 días)
- Auto-genera N `priceFormGroup()` con breakpoints pre-llenados
- Usuario solo llena los precios

### Input de tiempo inline
Reemplazar `variable-time` (picker flotante) por input `H:MM` + steppers ↑↓.
- Elimina el bug ngModelChange/setValue
- FormControl interno cambia a `number` (minutos)

### Virtual scroll
CDK Virtual Scroll obligatorio para 50+ filas. Requiere altura de fila fija.

### Formato de tiempo con días
Para tiempos ≥ 24h mostrar etiqueta secundaria: `48:00 = 2 días`.

## Deuda técnica pendiente

1. Extraer `parkedTimeToNumber` + `minutesToTimeString` a `core/utils/parked-time-utils.ts` + tests (duplicado en create y update).
2. Cambiar `PriceFormControls.fromParkedTime/toParkedTime` de `FormControl<string>` a `FormControl<number | null>`.
3. Migrar `variable-rate-form` de NgModule a standalone.
4. Reemplazar `variable-time` por input inline.
5. Implementar generador de tramos.
6. CDK Virtual Scroll en el FormArray.
7. "Add row" inline al pie, sin `[disabled]="!form.valid"`.

## Bloqueante conocido

`validations-settings/validation-form` importa `VariableTime` del NgModule. Debe migrarse en PR previo antes de poder eliminar `variable-time`.

**Why:** Israel quiere limpiar deuda técnica, mejorar UX, y soportar el caso real de 50+ tramos en estacionamientos de larga estancia.

**How to apply:** Cuando Israel quiera arrancar el refactor, los archivos clave son `rates-forms.d.ts`, `core/forms/settings/rates.ts`, `variable-rate-form/`, `create-variable-rate/`, `update-variable-rate/`, `variable-time/`. Documentación completa en `docs/variable-rate.md`.
