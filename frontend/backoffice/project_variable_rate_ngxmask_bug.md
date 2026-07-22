---
name: Variable rate — ngx-mask bug (parche merged, causa raíz pendiente)
description: El bug de #parkedTimeToNumber fue parcheado en PR #44, pero la causa raíz arquitectónica (strings con ngx-mask) sigue sin resolver.
type: project
originSessionId: a842525e-7ac3-42be-a7fa-5b793a2819fa
---
**Parche merged (PR #44).** La causa raíz arquitectónica sigue pendiente — ver `project_variable_rate_refactor.md`.

El bug de ngx-mask (`:` stripped → cálculo de minutos incorrecto) fue parcheado en PR #44 (`fix: correct variable rate time parsing when ngx-mask strips colon separator`). El parche hace que `#parkedTimeToNumber` maneje el string sin colon correctamente (last 2 chars = minutos, resto = horas).

Sin embargo, la causa raíz — que los campos de tiempo sean `FormControl<string>` procesados por ngx-mask — sigue en pie. El formato interno del FormControl sigue siendo ambiguo y la lógica de conversión está duplicada en `create-variable-rate` y `update-variable-rate`.
