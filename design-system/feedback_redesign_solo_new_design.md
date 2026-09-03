---
name: feedback_redesign_solo_new_design
description: "Todo el trabajo de rediseño va en el diseño nuevo. Una ruta con entrada única sirve al clásico, así que 'en sitio' siempre significa 'y al clásico también'"
metadata:
  node_type: memory
  type: feedback
  modified: 2026-09-03
---

Confirmado por Israel el **2026-09-03**, después de que yo lo incumpliera dos veces en la misma sesión:

> «Todo lo que tenemos que hacer respecto a lo del rediseño, va en el new design.»

La regla de tres puntos ya estaba en [[project_bo_module_migration]]. Esta memoria es sobre **por qué la incumplí de todas formas**, que es la parte que no era obvia.

## El test, y es de una línea

**Si el módulo tiene una entrada de ruta única, sirve al diseño clásico.** No hay otra lectura. `newDesignGuard` es lo único que separa los dos, así que sin ruta duplicada cualquier cambio «en sitio» se lo come el operador que nunca pulsó el toggle.

Antes de tocar un módulo: `grep` su path en `app.routes.ts`. Una entrada → no se toca. Dos con `canMatch` → el trabajo va en la de arriba.

## Las dos excusas que me parecieron válidas y no lo eran

1. **«El componente es un overlay global, así que es agnóstico del diseño.»** Lo usé para migrar los 22 snackbars de rates al toast del DS. Es agnóstico de **layout**, no de lo que ve el usuario. Un toast en la esquina en vez de un snackbar de Material es un cambio visible para todos.

2. **«El módulo ya se veía igual en los dos diseños, con hex a mano, así que pasarlo a tokens conserva esa propiedad.»** Lo usé para cambiar las KPI cards de trend por `cp-stat-card`. Es cierto y da igual: seguir siendo consistente entre diseños no autoriza a cambiar lo que ve el clásico.

**El patrón que comparten:** las dos razonan sobre si el cambio es *coherente*, cuando la pregunta es si es *visible para el clásico*.

## Qué sí se puede hacer sin módulo nuevo

Infraestructura **sin consumidores**. Se quedó en pie de aquella sesión y no toca ningún diseño:

- `core/services/notifier-service.ts` — la fachada del `NotificationService` del DS. No la llama nadie; es contra lo que se escribe el `ds-*`.
- `core/enums/api-error-code.ts` — los 12 códigos de error del backend.
- `core/i18n/general-services-i18n.ts` y `core/i18n/rates-i18n.ts` + los namespaces `GENERAL_SERVICES.ERROR` y `RATES` en `en.json`/`es.json`.

Claves i18n sin usar y un servicio sin llamadas son trabajo previo del rebuild, no rediseño.

Los arreglos **dentro de la librería** tampoco entran en esto: ahí no hay diseño clásico. La Fase 0 (CVA en checkbox/switch/radio, `cp-menu` al overlay del CDK, el CSS del CDK en el `styles.scss` de la lib) se quedó entera. Ver [[project_material_to_corepark_ui]].

## Cómo preguntar cuando dude

Preguntarlo **antes** del alcance de call sites y del i18n, que es lo que hice mal: pregunté esas dos y no la única que decidía. Si un cambio se ve en pantalla y el módulo tiene ruta única, la pregunta es «¿esto lo debe ver el clásico?», y la respuesta por defecto es no.
