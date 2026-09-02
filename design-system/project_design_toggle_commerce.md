---
name: project_design_toggle_commerce
description: "commerce: toggle entre diseño legacy y nuevo — initializer + canMatch + gating CSS opt-in + reload; qué rama va a prod"
metadata: 
  node_type: memory
  type: project
  originSessionId: 02d8be60-b4d7-4d73-99ae-4bda6564aa7f
  modified: 2026-08-31T22:06:39.906Z
---

Construido el **2026-08-31** en `~/Dev/frontend-commerce`. Los dos diseños **conviven** y el usuario elige; el cambio recarga la app.

## Ramas

- **`feature/design-toggle`** ← **la que va a producción.** Nació **de `main`** (requisito de Israel) y trajo el diseño nuevo mergeando `feature/auth-layout-ds`.
- `feature/staging` es entorno de pruebas y **sumidero** — ver [[feedback_branching_staging]].
- Ambas en `2026.8.31` (CalVer día real, ver [[project_calver_apps]]) y con `corepark-ui@0.0.29` del registry.

## Cómo se propaga la elección

1. **`designProvider`** (`core/init-providers/design.ts`) lee localStorage de forma **sincrónica** como app initializer y estampa `data-design="new|legacy"` en `<html>` **antes del primer render**. Mismo patrón que `colorSchemeProvider`, que ya existía. Nada asíncrono aquí: cualquier espera obligaría a pintar algo mientras resuelve, que es justo el destello que esto evita.
2. **La ruta raíz está declarada dos veces**, elegida por `canMatch` (`core/guards/design.guard.ts`): `DsAppLayout` si el toggle está activo, `MainLayout` si no. La entrada legacy **no lleva guard**, así que es el fallback — un valor corrupto o ausente aterriza en el diseño que siempre ha ido a producción. **El default es legacy**; el nuevo es opt-in.
3. Las dos entradas comparten `appRoutes`. Las páginas son las mismas, solo cambia el marco.
4. `/auth` va fijo a `DsAuthLayout`: **el login siempre usa el diseño nuevo**. Un usuario sin sesión no puede llegar al toggle, así que hacer que el login dependa de una preferencia que no puede cambiar sería confuso.

## Los headers de página: el gating es OPT-IN, y eso es lo importante

La migración al diseño nuevo había **borrado el header a mano de 31 plantillas**, porque el shell lo pinta desde la ruta. Para que el diseño legacy siguiera entero hubo que restaurarlos (Israel eligió restaurar en vez de enseñar a `MainLayout` a pintarlo).

Se gatean con **una regla global**, no con un `@if` en cada componente:

```scss
html[data-design='new'] .page-header--legacy { display: none; }
```

**`html[...]` por especificidad:** cada página define `.page-header` en su SCSS encapsulado, que compila a `.page-header[_ngcontent-x]` (0,2,0). El prefijo de etiqueta lo sube a 0,2,1 y gana sin `!important`.

### Empezó al revés y hubo que invertirlo

La primera versión fue `:not(.page-header--own)` — ocultar todo salvo lo que optara por salirse. **El default estaba mal**: una página que no supiera nada del toggle salía **sin título, en silencio**. Aparecieron 3 excepciones a mano, y una cuarta (`vehicle-info-links`) llegó desde otra rama y solo se detectó auditando.

Invertido: la clase marca **las 22 páginas cuyo título sí pinta el shell**. Cualquier otra conserva el suyo y no necesita saber nada. El peor caso para una página nueva pasa a ser un header con estilo viejo dentro del diseño nuevo: **feo y visible**, en vez de ausente e invisible.

**Regla general que deja:** cuando un default puede fallar en silencio, invertirlo hasta que el fallo sea visible.

### Las excepciones que quedan, y por qué

**2 páginas** (`access-control/roles`, `access-control/users`) pintan **su propio `cp-page-header`** porque proyectan acciones dentro, y un header pintado por el shell no puede recibir contenido proyectado. Llevan `@if (isNewDesign()) … @else …` en la plantilla; su bloque legacy solo existe bajo el diseño viejo, así que **no llevan la clase**.

**4 páginas sin marcar** conservan su header en ambos diseños porque el diseño nuevo no les da ninguno: `pms/connections` y `pms/arrivals-file` declaran `ownHeader: true`; `aggregators/connections/ocra` y `settings/vehicle-info-links` **no tienen `title` en su ruta**.

### Trampa al auditar esto

Mapear archivo de página → entrada de ruta por **nombre de carpeta** da falsos positivos: hay dos rutas `connections` (`pms/` con `ownHeader`, `aggregators/` sin él) y una ventana de regex se lleva el `ownHeader` del vecino. Me pasó **dos veces**. Verificar siempre la entrada de ruta concreta antes de creerse el resultado.

## El switch

**Opción de menú, no un slide toggle.** Israel lo pidió así, y llevó a algo mejor: **el switch de tema tampoco es un componente compartido** — cada shell escribe su propio botón en su idioma y solo el estado es común. El componente `design-toggle` que había hecho con `cp-switch` se borró.

- **Diseño nuevo:** `cpNavAction` dentro de `user-settings`, debajo del de tema → aparece en el rail **y en la hoja móvil** con una sola declaración.
- **Legacy:** un `.sign-out-btn` en el footer del sidebar (esa clase es la forma de "opción de menú" de ese sidebar), con el `palette-icon` que ya importaba.

Las etiquetas nombran el diseño al que **vas**, como el botón de tema. Ninguna es condicional: un shell solo renderiza bajo su propio diseño, así que la única dirección posible es la otra.

**Al pulsar:** overlay full screen → escribir localStorage → `location.reload()`.

- El overlay (`design-switch-overlay`) vive en **`app.component`, la raíz**, no dentro de un shell: los dos shells son **contextos de apilamiento** y un `position: fixed` declarado dentro no podría cubrir la página. Misma lección que el `cp-menu` del rail.
- Opaco y sin fade: un fade dejaría ver el diseño que se abandona.
- El reload espera **dos `requestAnimationFrame`** — el primero para que change detection pinte, el segundo para que el navegador lo ponga en pantalla. Antes de eso, recargar destruye la página con el diseño viejo aún visible.
- **Recarga en vez de intercambiar en vivo:** routing, propiedad del header y el tema de Material dependen del diseño, y un reload no puede aplicarse a medias.

## Sin verificar

**Nada de esto se ha visto en pantalla.** Y `pnpm test` de commerce **no arranca**: falta `jest-environment-jsdom`, ni declarado ni instalado, tampoco en `main`. Es preexistente.

Ver [[project_responsive_shell]] y [[feedback_angular_gotchas]].
