---
name: Material nativo y sin ::ng-deep
description: No pelear con los estilos internos de Angular Material. ::ng-deep está deprecado; los overrides de terceros van en parciales globales acotados por clase. Ante una fricción visual, usar el componente como viene antes que forzarlo.
metadata:
  type: feedback
---
Dos reglas que salieron de la misma sesión (2026-09-08, campo de expected departure en la guest page).

## `::ng-deep` está deprecado — no usarlo

Aunque haya precedente en el repo (`vehicle-info.component.scss` lo usa), no es justificación para código nuevo. Las salidas, en orden de preferencia:

1. **Los design tokens de Material**, que son API pública: `--mat-form-field-container-height`, `--mdc-outlined-text-field-outline-color`, `--mat-datepicker-toggle-icon-color`, etc. Heredan, así que se pueden poner desde el SCSS del componente sobre la clase del host. Sobreviven a que Material cambie su markup interno.
   **Verificar que el token existe** antes de usarlo — `grep -rhoE "\-\-mat-[a-z-]+" node_modules/@angular/material/` — en vez de inventar nombres.
2. **Un parcial global** en `src/assets/styles/_custom-*.scss` registrado en `styles.scss`, acotado por una clase del componente. Es la convención que ya usa `_custom-mat-dialog.scss`. Obligatorio para markup de terceros: un stylesheet de componente no alcanza a un hijo de librería sin `::ng-deep`. Si hay que ganarle a la hoja de la librería, poner el `@use` **después** del suyo y no usar `!important`.
   Acotarlo importa: el mismo `lib-country-list` se usa sin envolver en el flujo de request-car y ahí **sí** necesita su borde.

## Usar Material como viene, no forzarlo

Si un componente de Material se resiste, casi siempre es que se está usando a medias. El caso concreto: forzar el `mat-form-field` outlined a 48px con tokens dejaba un hueco muerto debajo del valor. Ese espacio es el del **label flotante**, y la solución no era reclamarlo sino poner un `<mat-label>`.

Israel lo dijo así: *"Pon los mat-form-fields y matInputs sin cambiar, usemos los nativos"*.

**Why:** un imitador de un componente del sistema de diseño se desincroniza en cuanto el tema cambia, y pelearse con el CSS interno de una librería se rompe en cada upgrade. Además la sobrecomplicación es señal de que el diseño va contra la corriente del componente.

**How to apply:** ante fricción visual con Material, primero preguntar qué parte del componente no se está usando. Si de verdad hay que ajustar, tokens; si el markup es de terceros, parcial global acotado; `::ng-deep` nunca. Y cuando la solución empiece a acumular hacks —`readonly` forzado, overrides de formato, tokens a medias— proponer partir el problema: dos inputs en vez de uno resolvió esta feature completa.
