---
name: project_design_toggle_validation
description: "validation: toggle legacy/nuevo replicando el del BO; el gate por operador 1 y 2, y por qué aquí sí hubo que revertir el puente de tokens entero"
metadata: 
  node_type: memory
  type: project
  originSessionId: 3c304b9d-9ff8-4ae9-aeb6-7e2f0a049f97
  modified: 2026-09-01T19:53:14.824Z
---

Construido el **2026-09-01** en `~/Dev/frontend-validation`, rama **`feature/design-toggle`** (nacida de `main`, con `feature/auth-layout-ds` mergeada encima). Mismo diseño que [[project_design_toggle_backoffice]]: gate por operador **y el login también cambia**.

**Empujado el mismo día.** `feature/design-toggle` está en `origin` (sin PR abierto) y **`develop` avanzó a la punta por fast-forward** (`f3bfb8d..3e24a86`) — la rama ya contenía todo develop, así que no hay commit de merge. Consecuencia: **el diseño nuevo y el toggle viven ya en `develop`**, no aislados; cualquier rama nueva los lleva. El default sigue siendo legacy, así que nadie ve un cambio hasta pulsar el toggle.

⚠️ **validation NO tiene `feature/staging`** — ni local ni en origin, al revés que el BO y commerce. Aquí la rama de integración es `develop`.

## Topología de la rama

`main` iba **43 commits por detrás de `develop`**, y `feature/auth-layout-ds` era `develop` + 1 commit (toda la migración del DS en uno solo, 29 archivos). Así que la rama trae los 43 de `develop` además del diseño — es inherente a mergear esa rama, no una decisión.

**Que la migración cupiera en un commit es lo que hizo el trabajo abordable**: la superficie a restaurar en legacy sale de `git show 7d8c033`.

## Lo que se copió del BO tal cual

`Design` enum, `setDesign`, `canAccessNewDesign`, `DesignState` (con `isNew` / `prefersNew` / `switching` / `stamp()`), `designProvider` **después de `operatorDataFactory`**, `design-switch-overlay` en la raíz y el reload tras dos `requestAnimationFrame`.

`ProdOperatorsEnum` de validation no tenía CorePark ni SecureParking (su universo era Curbstand 98, PCA 274, AllAboutParking 343). **Se añadieron `CorePark = 1` y `SecureParking = 2`** — decisión de Israel: los mismos dos que el BO.

Aquí el operador sale de `localStorage[userData].metadata.operatorCompanyId`, no de un JWT decodificado, pero es igual de síncrono. Se añadió el `computed` `operatorCompanyId` a `OperatorDataState`.

## Lo que se hizo distinto, y por qué

**`canMatch`, no shells con `@if`.** El BO necesitó componentes porque su ruta raíz lleva 46 hijas inline; aquí son 3, así que se extrajeron a `APP_CHILD_ROUTES` / `AUTH_CHILD_ROUTES` y cada entrada se declara dos veces. Solo la entrada del DS lleva guard → la legacy es el fallback. Es la forma de commerce.

**Hubo que revertir el puente de tokens entero.** El BO no tenía uno; validation sí: la migración reescribió `root.scss` para que sus ~40 tokens aliasaran `--cp-ui-*`. Sin revertirlo, el diseño legacy renderiza con la paleta del DS. El bloque `html[data-design='legacy']` de `root.scss` repone los literales.

### Tres detalles de ese revert que cuestan encontrar

1. **`initial` en una custom property = no definida** (valor garantizado-inválido). Es lo correcto para los 7 tokens que el puente *creó* (`--color-text-100`, `--color-grey-900`, `--color-accent-800`…): antes se usaban sin estar definidos, así que la declaración se caía y aplicaba el fallback o el heredado. Dejarlos definidos tiñe pantallas legacy que nunca tuvieron color ahí.
2. **En propiedades normales `initial` NO es "sin declarar"** — `color: initial` es negro, `background: initial` es transparente. Lo que se quiere es **`revert`**. Aplica al `body`, al `thead` de `.table` y al `.input-searcher`.
3. **`--font-display` / `--font-sans` y `--color-purple` NO se revierten**: los únicos que los leen son el kiosk y el valet dashboard, que viven en rutas con layout propio, **fuera del toggle**. Y sus valores viejos nombraban Poppins y Poltawski Nowy, cuyos `@font-face` y `.ttf` se borraron en la migración.

**`_custom-table.scss` y `_input-searcher.scss` se `@use`n desde SCSS de componente**, así que el bloque legacy sale encapsulado (`html[data-design=legacy] .table[_ngcontent-x]`, 0,2,1) y le gana a la regla base (0,2,0). Se verificó en el bundle: `e5f7f6` en `home` y `request-car`, el del buscador en `overnight-report`.

## `provideColorScheme()` solo bajo el diseño nuevo

**El fallo que esto evita:** `ColorSchemeService` cae a `prefers-color-scheme` del SO cuando no hay nada guardado. Registrado sin condición, una estación con el equipo en oscuro habría estampado `data-theme="dark"` sobre una app que **nunca tuvo tema y no ofrece forma de quitarlo** — el toggle vive en el footer del shell nuevo.

Se lee de storage (`prefersNewDesign()`), no de `DesignState`: el array de providers se construye antes de que exista inyector. Y además el bloque de Material oscuro se acotó a `html[data-design='new'][data-theme='dark']`.

## El switch, en el idioma de cada shell

- **Nuevo**: `cpNavAction` en `user-settings`, debajo del de tema → aparece en el rail y en la hoja móvil con una sola declaración.
- **Legacy escritorio**: botón de icono `palette` en `app-header`, que es de lo que ese header está hecho (chat + logout, `mat-icon-button`).
- **Legacy móvil**: una fila en la hoja `USER` de `mobile-nav`, junto a Sign out.

Las etiquetas nombran el diseño **al que vas**. Ninguna es condicional: un shell solo renderiza bajo su propio diseño.

## Sin verificar

**Nada se ha visto en pantalla.** Build verde y sin warnings, y el gating comprobado en el CSS y en los chunks emitidos — eso es todo. En particular:

1. Nadie ha pulsado el toggle: el overlay, el reload y el camino completo están sin ver.
2. Un operador **no elegible** no es verificable en dev (el gate está abierto fuera de producción).
3. El legacy móvil (`mobile-nav`) y el sign-in Material restaurado.
4. `validation` sigue **sin toggle en `develop`**: esta rama es la única que lo tiene.

**El método que funcionó en el BO sigue siendo el único que sirve aquí: capturas DEV y PROD lado a lado.** Yo puedo verificar que el CSS existe en el bundle; no puedo ver cómo se ve.

## El nav del shell nuevo iba corto (commit `7c22021`)

`DsAppLayout` listaba **seis** destinos y le faltaba `/valet-dashboard`: bajo el diseño nuevo la página solo se alcanzaba escribiendo la URL. Añadido al final, donde lo pone el rail clásico.

**Va gateado por `showValetDashboard()`, no suelto.** La ruta lleva `valetDashboardGuard`, así que un enlace incondicional devolvería a `/` a casi todas las estaciones. Por eso `routes` pasó de const a `computed`, y con eso lo heredan el rail y la hoja de `cp-app-nav-bar` a la vez.

**Regla general:** un destino del nav cuya ruta tenga guard necesita la misma condición en el enlace, o el nav miente.

## Las tres pantallas fuera del shell (commit `3e24a86`)

**Kiosk, valet service monitor y valet customer monitor tienen layout propio**, así que **no pasaron por la migración del shell** y seguían dibujadas para una página siempre clara. El diseño nuevo trae tema y ninguna lo seguía. Es la clase de hueco que deja migrar por shell: lo que no cuelga del shell no se entera.

Patrón usado: cada valor es un knob en `:host` con el color clásico, re-apuntado bajo **`:host-context(html[data-design='new'])`**. El legacy queda intacto. Compila a `html[data-design=new] [_nghost-x]` (0,2,1), así que gana.

Lo que estaba roto:

- **Las dos cabeceras de monitor eran `#202020` fijo con `#fff`.** Ahora toman superficie, borde y texto del rail del shell. Sus `section` pasan de superficie de tarjeta a **canvas** (`bg-app`), que es donde van tablas y tablero.
- **Rejilla de la tabla en `rgb(0 0 0 / 32%)`** — invisible sobre oscuro.
- **Los cuatro acentos del tablero kanban** aproximaban a mano los colores de estado del DS. Los `-soft` pasan a los tokens **`-alpha`**: al ser traslúcidos un solo valor sirve sobre columna clara u oscura; los pasteles fijos se volvían bloques casi blancos en oscuro.
- **El kiosk tenía la elevación invertida en oscuro**: la página leía `--color-grey` → `bg-80` y las tarjetas `--color-white` → `bg-50`, o sea tarjetas **más oscuras que el fondo**. A canvas.
- **El azul del kiosk** (`--color-primary-700` → escalón más oscuro de info) desaparecía sobre tarjeta oscura. Re-apuntado **una vez en el host del layout**: los cuatro `kiosk-*` heredan de ahí, así que una declaración cubre los 8 call sites. Sigue azul — corrección de contraste, no rebranding.

**Truco que vale recordar:** re-apuntar un token del puente en el `:host` de un layout arregla de golpe todos sus descendientes, sin tocarlos. El coste es que quien lea `kiosk-success-screen.scss` no ve que `--color-primary-700` está redirigido — por eso el comentario va en el layout, que es donde se busca.

## Pendiente que esta rama NO tocó

El versionado: validation sigue en `2026.8.0` con el script del contador, mientras commerce y BO usan el día real — ver [[project_calver_apps]].

Ver [[project_pending_work]], [[project_design_toggle_commerce]], [[project_migration_status]].
