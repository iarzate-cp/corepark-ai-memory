---
name: cp-time-picker — panel implementado en el DS
description: Componente cp-time-picker (panel) creado en design-system para reemplazar nxt-pick-datetime timer-only en backoffice; HH:mm string en 24h; falta wrapper CVA
type: project
originSessionId: dcd7e602-69f1-49b6-9846-55f5d103065d
---
`cp-time-picker` ya existe en `@corepark/corepark-ui` como panel autocontenido. Vive en `projects/corepark-ui/src/lib/components/time-picker/`. Documentado en `docs/time-picker.md`. Build verificado y syncedo a `frontend-backoffice/node_modules` (rsync `--delete --exclude=styles.css` desde `dist/corepark-ui/`).

**API:**
- `value: string | null` (`'HH:mm'` 24h, parser regex estricto, inválido → `09:00`)
- `hour12: boolean` (default `true`, AM/PM toggle UI)
- `minuteStep: 1|5|10|15|20|30|60` (default `5`, snap automático en cambio)
- `apply: output<string>` (`'HH:mm'` 24h zero-padded)
- `cancel: output<void>`

**Why:** Decidimos `'HH:mm'` string en 24h en vez de `Date` porque `nxt-pick-datetime` arrastraba una fecha "fantasma" del día actual que causaba bugs de zona horaria. String serializable, comparable, lo que el backend espera.

**How to apply:** Para usarlo en formularios sin el wrapper, hay que conectar manualmente (panel emite `apply`, callback hace `form.controls.X.setValue(value)`). El wrapper `cp-time-input` (CVA + CDK overlay) es la siguiente iteración para colapsar a 1 línea por campo.

**Migración aplicada en backoffice (branch implementation/corepark-ui):**
- Wrapper local `shared/components/time-field/` — CVA + CDK overlay + mat-form-field, envolviendo `cp-time-picker`. Patrón para sustituir cuando exista `cp-time-input` en el DS.
- 5 archivos HTML migrados a `<time-field>`: `overnight-rate-form` (billTime), `scheduled-rate` (startsAt + endsAt), `schedule` (8 instancias: 4 days × 2 + shifts), `automated-report` (startAt), `create-scheduled-report` (date).
- Form schemas: `FormControl<Date>` → `FormControl<string>` en `rates.ts` (billTime, startsAt, endsAt) y `overnight.d.ts` (date). Tipos en `rates-forms.d.ts` y `master-report.interfaces.ts` también actualizados.
- Submit logic limpiada en `create-overnight-rate`, `update-overnight-rate`, `create-flat-rate`, `update-flat-rate`, `create-variable-rate`, `update-variable-rate`, `create-scheduled-report`, `automated-report-dialog`, `schedule.component`: `formatDate(date, 'HH:mm:00')` → `${value}:00`, eliminado el truco de `new Date(yyyy-MM-ddT${time})`. Imports de `formatDate` removidos donde quedaron muertos.
- Build dev pasa en ~9s.

**Pendiente:**
- `cp-time-input` (CVA wrapper) en el DS — cuando exista, sustituir el wrapper local por el del DS.
- `cp-date-picker` + `cp-date-input` (single date+time) para los casos `update-shift` y `old-tip-report`.
- `cp-date-range-input` (wrapper sobre `cp-date-range-picker`) para colapsar las 40+ líneas de CDK overlay en `ticket-log-date-filter` y los 9 archivos en modo range.
- `period-control.component` aún usa `nxt-date-time` en modo range — depende de `cp-date-range-input`.
- Eliminación final de `nxt-pick-datetime` del `package.json`, `app.config.ts` (DateTimeModule/NativeDateTimeModule), y `styles.scss` (`@use 'nxt-pick-datetime/assets/picker'`) bloqueada hasta migrar todos los otros modos.
- Verificación visual en navegador NO realizada — recomendado validar antes de PR.
