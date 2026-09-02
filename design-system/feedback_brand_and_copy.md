---
name: feedback_brand_and_copy
description: la marca se escribe CorePark; títulos de página en title case; el patrón de barrido que respeta identificadores
metadata: 
  node_type: memory
  type: feedback
  originSessionId: aea67aa5-d32a-4d47-b014-93e055e5f1b9
  modified: 2026-08-29T00:14:36.675Z
---

Reglas de Israel, **2026-08-28**.

## 1. La marca es **CorePark**, no Corepark

**Why:** es el nombre correcto del producto y estaba mal escrito en 40 sitios de copy visible.

**How to apply:** al escribir cualquier texto de UI, i18n, subtítulo de ruta o plantilla de mensaje, siempre `CorePark`.

**Nunca tocar** los identificadores: `CoreparkIsotypeComponent`, `CoreparkIconComponent`, `CoreparkImagotypeComponent`, `CoreparkUi`, el paquete `@corepark/corepark-ui` y los selectores en minúscula.

El barrido se hizo con **`Corepark(?![A-Za-z])`** — el lookahead negativo respeta todos los identificadores por construcción. Recuento tras aplicarlo: 0 en los cuatro repos.

Incluyó **datos**, no solo etiquetas: el seed `{ name: 'CorePark' }` y la plantilla de SMS `'Thank you for using CorePark…'` de commerce. Si algún día eso refleja lo que hay en base de datos, hay que revisarlo.

**Sin resolver**: la pestaña dice `CorePark - BackOffice` y el i18n `CorePark Backoffice`. Israel pidió la mayúscula de CorePark, no la de BackOffice — no inventar la segunda.

## 2. Títulos de página en **title case**

**Why:** son el nombre canónico de la página y aparecen en el `<h1>` y en la pestaña; el copy original de commerce mezclaba sentence case y title case.

**How to apply:** `Valet App Config`, no `Valet app config`. Se corrigieron 9 títulos de ruta y **10 etiquetas del rail** de commerce, que estaban en desacuerdo con ellos.

Asimetría deliberada que se conserva: el rail dice `Valet App` y la página `Valet App Config` — dentro del grupo Settings el "Config" sobra en el menú pero no en la pestaña.

## 3. El copy de producto no se inventa

**Why:** escribir texto visible que nadie ha aprobado es tomar una decisión de producto.

**How to apply:** cuando falte un subtítulo, o se pide, o se **redacta leyendo lo que la pantalla hace de verdad** y se entrega como borrador explícito. En el Partner los tres subtítulos salieron de leer los formularios y las columnas de las tablas, y se entregaron marcados como borradores.

Ver [[project_route_titles]].
