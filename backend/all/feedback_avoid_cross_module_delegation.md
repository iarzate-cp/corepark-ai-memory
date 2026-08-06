---
name: No delegar entre módulos por reuse "menos invasivo"
description: Cuando un módulo nuevo necesita una operación parecida a la de otro módulo, es tentador inyectar el service ajeno y delegar; casi siempre arrastra constraints y semántica que no aplican
type: feedback
---
## Regla

Cuando estás construyendo un endpoint nuevo bajo un módulo A y notas que existe algo similar en un módulo B (ej. `com.corepark.partner.hotel.*`), **no** lo reutilices inyectando `IHotelService` desde el módulo A "para no invadir". Escribe DAO/service propios del módulo A, aunque implique duplicar 20-30 líneas de SQL/Java.

**Why:** el reuse por delegación arrastra:
1. **Constraints del caso de uso original**. Ej. `HotelDao.editTicketInfo` exige `check_out IS NULL` (el hotel edit era solo para tickets activos). Cualquier consumer nuevo hereda ese filtro aunque no lo necesite.
2. **Efectos secundarios inesperados** (MQ publishes, audit rows, event dispatches) que puedes no querer replicar.
3. **Mismatch de shapes entre paquetes**. Ej. `com.corepark.partner.commons.beans.response.ResultBean` vs `com.corepark.valet.commons.beans.response.Response<T>` — dos maneras distintas de envolver códigos y errores. Hay que adaptar.
4. **Mismatch en enums homónimos entre paquetes**. `com.corepark.partner.enums.MessageCode.BUSINESS_TICKET_WITHOUT_REGISTERED_CHECK_IN` code string = `"TICKET-WITHOUT-REGISTERED-CHECK-IN"` (guiones). `com.corepark.valet.utils.enums.MessageCode.BUSINESS_TICKET_WITHOUT_REGISTERED_CHECK_IN` code string = `"TICKET-WITHOUT_REGISTERED_CHECK_IN"` (guiones **y** underscores mezclados). `MessageCode.findByName(code)` no encuentra el equivalente y cae en el fallback `ERR_NOT_IMPLEMENT` (501/`MSG-NOT-FOUND`). Cualquier business error del módulo ajeno se degrada silenciosamente.

**How to apply:**
- Regla operativa: si el service B pertenece a un paquete conceptualmente distinto (partner/hotel vs valet/validations), **construye la operación en el módulo propio** aunque sea "más código". Es menos invasivo *en el sentido real*: sin acoplamiento entre paquetes y sin arrastrar behaviors ajenos.
- Excepción legítima: si el service B es explícitamente compartido (`com.corepark.valet.commons.*`, `com.corepark.valet.utils.*`), reusa sin culpa.

**Caso 2026-08-06 (edit ticket info del portal de validación):**
- Primer intento: `ValidationsService.editTicketInfo` inyectaba `IHotelService` y delegaba. Arrastró: MQ Rabbit sync, filtro `check_out IS NULL` (no queríamos ese constraint), y mapeo de codes entre `partner.MessageCode` y `valet.MessageCode` que estaba roto (guiones vs underscores). Devolvía 501 MSG-NOT-FOUND para cualquier caso de negocio.
- Refactor: DAO propio en `ValidationsDao` con SQL propio (SELECT + UPDATE), sin `check_out IS NULL`, MQ publish propio en `ValidationsServiceImpl`. Todo en el paquete `com.corepark.valet.validations.*`. Simple, sin arrastres.
