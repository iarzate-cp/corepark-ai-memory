---
name: microservice-reports se renombró a ms-reports-service en GitHub
description: El repo local apunta al nombre viejo y funciona por redirect; GitHub avisa en cada push. Actualizar el remote cuando se pueda.
type: reference
---

El repo de GitHub `corepark/microservice-reports` fue renombrado a **`corepark/ms-reports-service`** (alineado con la nomenclatura de los demás: `ms-valet-service`, `ms-backoffice-service`, etc.).

La working copy local en `~/Dev/Back-End/microservice-reports/` sigue con el remote viejo. **Funciona** — GitHub redirige — pero avisa en cada push:

```
remote: This repository moved. Please use the new location:
remote:   git@github.com:corepark/ms-reports-service.git
```

Para actualizarlo (respetando el host alias `github.com-work` que usa esta máquina):

```sh
git remote set-url origin git@github.com-work:corepark/ms-reports-service.git
```

Detectado 2026-08-13. **Todavía no se aplicó** — el usuario no lo ha pedido.

Ojo al construir URLs de PR: el link que devuelve GitHub en el push ya usa el nombre nuevo (`https://github.com/corepark/ms-reports-service/pull/new/...`), aunque el directorio local se siga llamando `microservice-reports`.
