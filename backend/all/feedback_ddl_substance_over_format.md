---
name: DDL alignment — substance over ceremonial format
description: When writing or comparing DDL scripts against a reference, prioritize substantive documentation (Purpose block, COMMENT ON *, rollback rationale) over ceremonial boilerplate parity (psql \set headers, exact rollback numbering style, block delimiter counts).
type: feedback
---

Al escribir o comparar un DDL contra otro tomado como estándar (ej.
`DDL_RELEASE_171_EV_CHARGING_MODULE.sql`), quédate en las cosas
importantes y no en la ceremonia de formato.

**Sí importa (substance):**
- Bloque `/* Purpose */` que explique el problema, alcance, prerequisites,
  safety, rollback strategy, decisiones no-obvias.
- `COMMENT ON TABLE / COLUMN / CONSTRAINT / INDEX` correctos, con el estilo
  del ddl_style_guide (present tense, lifecycle inline, business "why"
  cuando aporta, sin historia/env/team internals).
- Rationale explícito de decisiones de diseño en el `Purpose` o en
  comentarios in-line.

**No importa (ceremonial):**
- Presencia/ausencia del bloque `\set vDatabase corepark` / `\c :vDatabase;`
  al inicio.
- Exactitud de la numeración `-- 1. …` en el bloque ROLLBACK vs prosa.
- Conteo exacto de asteriscos en los delimitadores de bloque.
- Presencia de un bloque `GRANT PRIVILEGES` vacío cuando el script es
  data-only y no crea objetos.

**Why:** Israel lo dijo explícitamente tras proponerle 3 alineaciones al
DDL de coupon_request backfill (2026-07-22): "no es necesario, sólo
seguir cosas importantes como los comentarios en dado caso de que se
necesiten". El reviewer se enfoca en el fondo; si tiene preferencias de
forma las pide.

**How to apply:** Al comparar contra un DDL de referencia, reporta las
diferencias en substancia (falta un comentario clave, ausencia de una
nota de safety, ausencia de un GRANT que debía estar) y menciona las
ceremoniales solo como opcionales. No refactorices para alcanzar 100%
paridad visual — es trabajo que probablemente el reviewer no pedirá y
que sesga la conversación hacia forma en vez de fondo.
