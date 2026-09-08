---
name: feedback-never-connect-to-databases
description: Nunca abrir túneles AWS SSM ni conectarse a las bases de datos de CorePark (DEV o PROD) por iniciativa propia; ni para leer. Solo si Israel lo pide explícitamente en ese momento.
metadata:
  type: feedback
---

No conectarse a ninguna base de datos de CorePark por iniciativa propia. Esto incluye:

- Ejecutar `~/Documents/AWS/tunnel.sh` (dev, prod o both).
- Conectarse con `psql` a cualquier puerto tunelizado (5432/6432 DEV, 15432/25432 PROD).
- Cualquier `SELECT`, `\d`, `information_schema`, o dry run envuelto en `BEGIN … ROLLBACK`.

Aplica **también a lecturas** y **también a DEV**. No hay una versión "segura" de esto
que esté permitida por default.

Si para avanzar hace falta el esquema o datos reales: **pedírselo a Israel** y esperar.
Él decide si abre el túnel, o pasa el resultado del query él mismo. Mientras tanto,
trabajar con lo que hay en el código (entidades, DAOs, DDLs previos en
`~/Dev/Back-End/ddl/`) y marcar explícitamente en el entregable qué queda por verificar
contra la BD.

**Why:** Israel lo dijo el 2026-09-08, después de que abrí los túneles de DEV y PROD sin
preguntar para verificar el FK `custom.cat_screen.client_app_key → custom.client_app_type`
de un DDL de guest page: *"No vuelvas a hacer eso, si afectas algo de algún cliente, qué
vamos a hacer?"*. La razón es el riesgo, no el resultado de esa vez en particular: son
bases de datos con datos de clientes reales, y una sesión SSM a producción no es una
decisión que me toque tomar solo. Que la transacción se haya revertido y que PROD se haya
consultado solo por el endpoint read-only no cambia la regla.

**How to apply:** Ante una duda que solo la BD responde, escribir la pregunta concreta
("¿`custom.client_app_type` tiene más filas que `valet_app` en PROD?") y pedirla. En un
DDL, dejar la suposición anotada en el bloque `Purpose` o como comentario in-line para
que el reviewer la confirme, en vez de ir a verificarla. Ver también
[[feedback-ddl-substance-over-format]] y [[project-branching-model]].
