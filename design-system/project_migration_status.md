---
name: estado-de-la-migraci-n-de-layouts-al-ds-2026-08-26
description: "qué está hecho y qué falta por repo, ramas activas, y el riesgo del build local por delante del registro"
metadata: 
  node_type: memory
  type: project
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-31T20:51:02.343Z
---

Sesión del **2026-08-26**. **Histórico** — el estado que describe quedó cerrado el 2026-08-31: se publicó `0.0.29`, los tres consumidores la instalaron del registry, y todo está commiteado y empujado. Ver [[project_pending_work]] para el estado real y [[project_responsive_shell]] para lo que vino después.

## design-system — rama `develop`

Versión `0.0.28` **publicada** en GitHub Packages. Pero el `dist/` local y los `node_modules` de las apps tienen un build **posterior** con: gráficos oscuros del auth, `--decor-alt`, transiciones `@property`, `cp-app-layout`, `cp-app-nav`, `cp-app-nav-footer`, `NavActionDirective`, `ColorSchemeService`, `provideColorScheme`, tokens `_breakpoints`/`_transitions`, y todos los ajustes finos del menú.

⚠️ **RIESGO — RESUELTO el 2026-08-31.** Decía: los tres consumidores declaran versiones del registro pero su `node_modules` lleva el build local, y un `pnpm install` lo revierte. Se cerró publicando `0.0.29` y apuntando los tres a esa versión exacta. **El patrón sigue vigente para cualquier build local sin publicar** — ver [[feedback_pnpm_add_borra_el_sync]].

### Sync del 2026-08-27 — verificado

Tras mover las directivas a `lib/directives/` (cambia las rutas `@use` de `styles.scss`), se sincronizaron **commerce, validation y backoffice**. Verificado en los tres: `directives/{button,ripple}` presente, `styles.scss` apuntando a las rutas nuevas, 8 selectores `.cp-btn*` en el CSS y `AppLayoutComponent` en los typings. **Los tres compilan con `ng build`** — es la única verificación real de que los cambios de API (subpaths `pipes`/`utils` eliminados, `COLOR_PRIMARY_DARK`→`DARKER`) no rompieron nada.

Comprobación previa que vale repetir: los cuatro consumidores definen sus propios `--color-*` en su `_root.scss` y su DS instalado ya solo emitía `--cp-ui-*`, así que el sync no tocaba el namespace de tokens.

Sin commitear: `layouts/`, `services/`, `components/app-nav/`, tokens nuevos, fix del isotipo, CLAUDE.md, demo pages (`app-layout`, `auth-layout`), `publish:lib`, scripts a pnpm.

## frontend-validation — rama `feature/auth-layout-ds` (desde `develop`, NO main)

`develop` iba 43 commits por delante de `main`. **Hecho:**
- `/auth` → `DsAuthLayout`; sign-in migrado a `cp-form-field`/`cpInput`/`cpButton`/tabler
- `/` → `DsAppLayout` con logo, `cp-app-nav` (6 links planos), footer con Chat + tema + Sign out
- **Puente de tokens en `root.scss`**: los ~40 tokens locales aliasan a `--cp-ui-*`; los 303 usos en 42 archivos no se tocaron. Se arreglaron 8 tokens que se usaban sin estar definidos
- `body` a `--cp-ui-color-bg-app`; `_custom-table.scss` y `_input-searcher.scss` desharcodeados
- **Tema oscuro de Material** bajo `[data-theme='dark']` con `mat.all-component-colors`
- `provideColorScheme()` del DS registrado
- `product="Partner"`

**Los layouts viejos `main-layout` y `auth-layout` NO se borran** — decisión de Israel del 2026-08-27. Ya no están ruteados, pero se quedan. **No volver a proponer borrarlos.** Son de los que más usan `--fw-black`, así que cuentan para los call sites del puente de tipografía.

**Pendiente:** los 3 fondos negros hardcodeados de `main-layout/ui/`.

### Tipografía — ajustada el 2026-08-27

El puente colapsaba dos pares con pérdida y aplanaba la app: `--fw-bold` (24 usos) y `--fw-black` (23) caían ambos en 700, y `--fs-small` (22) con `--fs-extra-small` (9) ambos en `paragraph-sm`. Se resolvió **extendiendo la escala del DS** en vez de reescribir los 47 call sites: `--cp-ui-font-weight-black: 900` y `--cp-ui-font-size-paragraph-xs: 0.625rem` (ver [[project_corepark_ui_tokens]]). El puente ahora mapea 1:1.

También se borró peso muerto: `_fonts.scss` (4 `@font-face` de Poppins y Poltawski Nowy que nadie referenciaba, uno apuntando a un `.ttf` inexistente), su `@use` en `styles.scss`, y `src/assets/fonts/`. **Los assets del deploy bajan de ~1.1 MB a 364 KB** — `src/assets` se copia entero, así que viajaban sin usarse.

**Lo que NO se tocó y por qué:** `_normalizer.scss` fuerza la familia con `* { font-family }`. Parece un smell pero es benigno: especificidad 0, cualquier regla de clase le gana, y el `!important` de icomoon es contra extensiones del navegador, no contra el `*`. Moverlo a `body` rompería los controles de formulario, que no heredan `font-family`.

## frontend-commerce — rama `feature/auth-layout-ds`

**Hecho:** `/auth` → `DsAuthLayout`; `/` → `DsAppLayout` con `product="CRM"`, menú de 1 link + 7 grupos sin ruta propia, footer con OTP + tema + Sign out; sign-in migrado a componentes del DS; `one-time-password` re-pielado con `cpNavAction`.

**Usa su `ColorSchemeState` propio**, NO el `ColorSchemeService` del DS — commerce ya tiene su app initializer y dos dueños del atributo `data-theme` se pelearían. Unificar es un paso pendiente (toca `valet-layout` y `auth-layout`).

**Pendiente:** `.npmrc` con un **PAT de GitHub en texto plano y trackeado en git** — rotar y sacar del control de versiones.

## frontend-backoffice — rama `feature/app-layout`

### Canvas al claro y `cp-menu` — 2026-08-27

**El canvas del contenido pasó de casi negro a `--cp-ui-color-bg-app`.** El `body` estaba en `--color-grey-500`, que es `hsl(0 0% 13%)` — un tono que esta app usa como **color de texto en 15 sitios**. La paleta local va de `--color-grey: 96%` a `--color-grey-700: 7%`: el 500 es un tono de texto, y usarlo como fondo de página era un accidente histórico, no diseño. Se quitó también el override `--cp-ui-app-layout-main-bg: transparent` que lo preservaba.

Rail (`bg-60`, #FAFAFC) y canvas (`bg-app`, #F5F6FA) quedan a un pelo, separados por el borde de 1px — es lo que definen los knobs del DS y cómo se ven commerce y validation. Las tarjetas del dashboard traen `background: white` + `box-shadow` propios, así que siguen leyéndose.

**`MatMenu` → `cp-menu`** en el selector de idioma. Con eso **no queda Material en el rail**.

- `[cpMenuTrigger]="langMenu"` toma la **instancia** del componente (`input.required<CpMenuComponent>()`), así que basta la ref de template `#langMenu`.
- **Sin `cpMenuPosition`**: el trigger está al fondo del rail y el auto-posicionamiento resuelve `top-start` solo. Verificado: panel a 215×82 abriéndose hacia arriba.
- El panel es `position: fixed`, no un overlay del CDK. Hay que comprobar que ningún ancestro cree bloque contenedor (`transform`/`filter`/`perspective`) o el `fixed` se posicionaría contra él. En el rail no hay ninguno.
- **Gap del DS:** el `disabled` de `cp-menu-item` hace `stopPropagation()`, que **no** bloquea otros listeners del mismo elemento — el `(click)` del consumidor sí dispara en un item deshabilitado. Se blindó con un early return en `onChangeLang`. Vale arreglarlo en el DS.

**Verificado con CDP** (clic real, no solo captura): abrir el menú, clic en "Español", y el nav se retraduce en vivo — `["Dashboard","Analytics",…]` → `["Tablero","Analíticas","Ubicaciones","Empleados","Conf. Clientes","Configuraciones","Reportes","Pagos"]`. Eso prueba la cadena entera, incluida la señal de `onLangChange` que alimenta el `computed` de rutas.

### Menú y footer del rail al DS — 2026-08-27

**El backoffice usa ya `cp-app-nav` y `cp-app-nav-footer`, igual que commerce y validation.** Antes había importado solo el chasis y dejado dentro `app-nav` y `user-settings` locales — eso era la migración a medias, y es lo que se veía distinto.

Lo que hubo que resolver, porque no era drop-in:

1. **Las etiquetas son claves i18n** (`NAVIGATION.DASHBOARD`). `cp-app-nav` renderiza `route.label` en crudo, sin pipe de traducción. Hay que traducir antes de pasar, y **reaccionando al cambio de idioma**: `toSignal(translate.onLangChange)` leído dentro del `computed` de las rutas, porque `translate.instant()` es una llamada normal, no una señal — sin eso las etiquetas se congelan en el primer idioma cargado.
2. **`urlTree` → `children`**, misma forma con otro nombre. La conversión vive en `app-nav-routes.ts`.
3. **Iconos.** `cp-app-nav` toma `icon?: TablerIcon`; el backoffice tenía un componente `app-nav-icon` con un `@switch` sobre sus SVG propios. Sustituido por un mapa ruta → icono Tabler.
4. **`BROWSABLE_MODULE_ROOTS`.** El acordeón del DS **navega al primer hijo** al abrir un grupo, así que Locations y Payments —cuyo root es una página real— se volverían inalcanzables desde el rail. Se resuelve alimentando `cp-app-nav` con **`getAppNavRoutesMobile()`**, que ya prepone el root como primer hijo justo para eso. El nombre "Mobile" engaña: esa variante es la correcta para el acordeón del DS.
5. **`user-settings` mapea 1:1** a `cp-app-nav-footer` (`heading`/`name`/`email` + acciones proyectadas con `cpNavAction`) — el propio DS documenta que ese componente se portó desde aquí.

**`app-nav` y `user-settings` NO quedaron muertos:** `app-nav-mobile` los usa dentro, así que siguen siendo la implementación de la hoja inferior en móvil. Commerce y validation no tienen slot `cpAppMobile`, así que ahí no hay nada del DS a lo que converger.

**Queda Material en el rail:** solo el `MatMenu` del selector de idioma.

**Bug que costó tres capturas:** al reescribir el SCSS del wrapper perdí `--imagotype-color`, y `corepark-imagotype` cae a su fallback `var(--color-white)` — la mitad "core" del wordmark se pintaba **blanco sobre el rail claro** y solo se veía "park". Parecía un recorte del logo, no un problema de color.

**Lección de proceso:** antes de esto estuve tres iteraciones empujando `--cp-ui-app-layout-brand-height` a 7rem y el padding del footer a `spacing-6` para que el layout del DS **imitara** al viejo píxel a píxel. Eso era usar los knobs para deshacer la migración. La pregunta correcta era "¿commerce y validation usan componentes del DS aquí?" — sí, y con eso la geometría cuadra sola.

### `cp-app-layout` consumido de la lib — 2026-08-27

El `app-layout` local (el original del que se **extrajo** `cp-app-layout`) ahora consume la versión del DS. Su `<aside>`/`<main>` propios se fueron; queda como wrapper con el estado (custom branding, catálogo, loader) y `product="BackOffice"`.

La geometría coincidía exacta, así que no hubo ajuste visual: `--aside-width: 15.5rem` = `--cp-ui-app-layout-aside-width`, `--main-padding: 4rem` = `--cp-ui-app-layout-main-padding`, y el `#fafafc` hardcodeado del aside **es** `--cp-ui-color-bg-60` en claro.

**El truco de los slots: `display: contents`.** La proyección no atraviesa `@defer` (mismo problema que `@if`, ver [[feedback_angular_gotchas]]), y este layout defiere logo, nav y user-settings. La solución es un `<div class="slot" cpAppBrand>` estático que envuelve el `@defer`, con `.slot { display: contents }` — lleva el atributo para la proyección en compilación pero no aporta caja, así que el componente diferido sigue siendo el hijo flex directo que el rail espera.

**Dos bugs que costó encontrar:**

1. **`app-nav` tenía `height: calc(100% - 21.5rem)`**, escrito para cuando era hijo directo de un aside de `100dvh` y descontaba a mano los bloques de marca y usuario. Dentro de `.cp-app-layout__nav` (que ya es `flex: 1 1 auto` entre esos dos bloques) restaba de más: la lista salía **recortada a media altura con un hueco muerto** hasta el footer. Ahora `height: 100%`, y fuera el `margin-top: 2rem` (el bloque de marca del DS ya trae su min-height).
2. **Choque de breakpoints.** El DS oculta el rail a `<1024px` (`$laptop`, 64rem) y no es un knob — es un media query, el consumidor no lo puede cambiar. La app usaba **1030** en tres sitios (`AppLayoutState`, `app-nav`, `app-nav-mobile`), o sea 6px donde se veían el rail y el nav inferior a la vez. Los tres a 1024.

**Renombrado `isBrowser` → `isDesktop`** en `AppLayoutState` (+ 3 consumidores): medía ancho de ventana, no entorno de ejecución.

Verificado renderizado en desktop y móvil poniendo una probeta temporal en `regularAccountGuard` (ya retirada) — es la única forma de ver el shell, que vive tras el login. **El fallo de proyección es silencioso**, así que aquí la verificación visual no era opcional.

`MainLayout` ya no se rutea pero sigue listado en `app.config.ts` (esa lista de `providers` es cruft, nadie los inyecta).

### Auth sin NgModule, todo standalone — 2026-08-27

`src/app/pages/auth/` no tiene **ni un `@NgModule`**. Las 3 páginas quedaron con la misma forma: `*.component.{ts,html,scss,spec.ts}` + `*.imports.ts` + `index.ts`.

- Borrados: `sign-in.module.ts`, `sign-in-routing.module.ts`, `config-new-password.module.ts`, `config-new-password-routing.module.ts`, y `recovery-password-routing.imports.ts` (mal nombrado — no tenía nada de routing).
- **Sin `standalone: true` explícito**: es el default desde Angular 19. Los `standalone: false` se fueron con los módulos.
- Los `index.ts` de sign-in y config-new-password exportaban el **módulo** (`ConfigNewPasswordModule as ConfigNewPasswordPage`, un alias que mentía); ahora exportan el componente.
- Rutas: 2 × `loadChildren` → `loadComponent`.
- Los imports viven en `*.imports.ts`, que es el patrón de la casa en el backoffice (`user-settings.imports.ts`, `app-layout.imports.ts`). Se aprovechó para quitar lo que no se usaba: ni `FormsModule` (los templates usan `[formGroup]`/`formControlName`, nunca `ngModel`) ni `CommonModule` (todo el control flow es `@if`).

**Trampa que costó un intento:** al pasar `config-new-password` de `loadChildren` a `loadComponent` quedó con `component: DsAuthLayout` **y** `loadComponent` en la misma ruta, y Angular no admite ambos. Hay que meter la página en `children: [{ path: '', loadComponent }]`. Con `loadChildren` no se notaba porque el módulo traía su propia ruta hija. **Y el `ng build` pasa igual** — Angular valida la config de rutas en runtime, así que esto solo se ve navegando.

Los 3 spec files usaban `declarations: [Component]`, inválido para standalone (el de recovery-password ya estaba roto de antes, porque ya era standalone). Pasados a `imports`. Ojo: **el backoffice no tiene script de `test`**, así que ningún spec se ejecuta.

Falta en el resto de la app: **61 archivos con `@NgModule`**, **50 componentes con `standalone: false`** y 3 rutas con `loadChildren`.

### Formularios de auth: Material → DS — 2026-08-27

Las **3 páginas** (`sign-in`, `recovery-password`, `config-new-password`) pasaron de `mat-form-field`/`matInput`/`mat-icon`/`mat-icon-button`/`actioner-raised` a `cp-form-field` + `cpInput` + `cpButton`, replicando la estructura del login de commerce: `h2` + `.subtitle` + `.form-fields` + submit a todo el ancho + enlace secundario centrado debajo.

- **`src/assets/scss/auth-forms.scss`** es el único SCSS: las 3 páginas hacen `@use 'auth-forms'`, así que un archivo las estiliza todas. Reescrito sobre `--cp-ui-*` directo, sin pasar por los alias locales `--fs-*`/`--color-grey-*`.
- **`sign-in-material.module.ts` borrado** y `SignInMaterialModule` fuera del módulo. `MatSnackBarModule` ya está en `app.config.ts` vía `importProvidersFrom`, así que inyectar `MatSnackBar` no necesita import local.
- **`sign-in.imports.ts` borrado** — nadie lo importaba, era un array de Material muerto.
- El patrón de errores viene de commerce: `cp-form-field` toma **un solo** string de error, así que los `@if` anidados por tipo de error colapsan en un `errorFor(control, kind)` con un mapa `FIELD_ERRORS`. **Método, no `computed`**: los controles son `FormControl` planos, no señales, y un computed nunca recomputaría.
- `hide`/`hideConfirm` de `config-new-password` pasaron de `boolean` público a `signal`.

**Bug arreglado de paso:** el template de `recovery-password` usaba `routerLink` pero su archivo de imports **no incluía `RouterLink`** — Angular admite atributos desconocidos en elementos nativos, así que ese enlace estaba muerto en silencio.

**Queda solo `MatSnackBar`** en las 3 (el servicio, no la UI del formulario). Dimensión de lo que falta en la app: 495 archivos importan Material, encabezados por `dialog` (327) y `snack-bar` (234).

### `cp-auth-layout` migrado — 2026-08-27

Wrapper nuevo en `shared/layouts/ds-auth-layout/`, mismo nombre y patrón que commerce y validation. Las **2 entradas de ruta** de `app.routes.ts` (`config-new-password` y `auth`, que cubren 3 páginas: sign-in, recovery-password, config-new-password) apuntan ya a `DsAuthLayout`.

Configuración y por qué:
- **`[showThemeToggle]="false"`** — el backoffice no tiene estado de color-scheme ni escribe `data-theme` (ver [[feedback_backoffice_theme]]), así que el panel renderiza en **claro**. Commerce y validation corren oscuro: es una diferencia visible entre apps, deliberada.
- **`[loading]` NO cableado** — existe `AppLoaderState`, pero su `isLoading` arranca en `true` y solo baja cuando arranca el shell autenticado. Cablearlo taparía el formulario con un spinner que nadie descarta.
- **`footerText` = la versión del `package.json`** (`import pkg from '@package'`), conservando que la pantalla de auth muestre el build — soporte lo pide.
- **`headline` = 'Welcome back'**, el copy que ya había. El panel está diseñado para una frase de producto de dos líneas (commerce: `'Smart parking.\nSimplified.'`, validation: `'Validation,\nwithout the friction.'`) y aquí no hay subheadline, así que **falta escribir copy de verdad**.

Verificado renderizado en `/auth/sign-in` y `/auth/recovery-password` (son pre-login, sí se pueden ver). Los formularios son `width: 100%` vía `auth-forms.scss`, así que encajan en la columna de 24rem sin tocarlos.

**Queda sin usar, no borrado** (por la política de no borrar layouts viejos): `shared/layouts/auth-layout/` completo con su `ui/iso-type/`, y `assets/images/auth-layout-bg.webp` (145 KB). Y en `app.config.ts` la lista de `providers` sigue con `MainLayout` y `AuthLayout` — nadie los inyecta, es cruft; la migración del `AppLayout` tampoco la tocó ni añadió `AppLayout` ahí.

**Ya no está sin tocar** (dato corregido el 2026-08-27). Su `package.json` declara `0.0.25` pero su `node_modules` lleva el build sincronizado. Es el origen de `cp-app-layout` y `cp-app-nav`. Pendiente: cambiar a Roboto Flex, y decidir qué hacer con `operator-logo` (marca blanca, 12 logos) y `app-nav-mobile` (manipula el DOM a mano).

## frontend-valet-web — rama `develop`, en `^0.0.21` — FUERA del sync

**Decisión confirmada por Israel el 2026-08-27: valet-web se queda fuera. No proponer sincronizarlo.**

Siete versiones por detrás y no forma parte de la migración. Su `node_modules` tiene una copia limpia del registro, así que no necesita refresco. Un `diff -rq` contra el `dist` daba 27 diferencias: le faltan `components/app-nav` y `components/breadcrumb` enteros, aún tiene `components/button` (en el dist las directivas ya viven en `directives/`), y difieren los SCSS de `date-range-picker` y `form-field`.

Sincronizarlo metería cambios visuales del select, `cp-table` y el date-picker en una rama que no los espera. Si algún día se hace, es `pnpm run sync:valet`.

## Tipografía — decisión tomada

**Todo a Roboto Flex**, la familia del DS. Es variable (`wght@100..900` continuo), así que los 5 tokens de peso se renderizan exactos — Lato no tiene 500/600, Inter no tiene 100/900.

Commerce, validation **y el backoffice (2026-08-27)** cargan `family=Roboto+Flex:opsz,wght@8..144,100..900` y su reset usa `font-family: var(--cp-ui-font-family)`. Ninguna app declara familia propia.

### Por qué importaba en el backoffice

Cargaba `family=Lato:wght@100;300;400;700;900` — **stops fijos, y Lato no tiene 500 ni 600**. El backoffice pide peso 500 en 26 sitios y 600 en 34, más los ~50 de los componentes del DS. Todos caían al más cercano o se sintetizaban: la jerarquía medium/semibold simplemente no renderizaba. Solo había dos referencias a `'Lato'` (el `<link>` y `_normalize.scss`), así que el cambio fueron dos líneas.

**Regla:** pedir siempre el eje como rango (`100..900`), nunca como lista de stops. Una lista reintroduce exactamente este bug.

### Deuda que introduce Roboto Flex: la cursiva

**Roboto Flex no tiene eje `ital`** y no trae compañera cursiva en su versión variable — Lato sí tenía cursiva real. Los **16 `font-style: italic`** del backoffice pasan a oblicua sintética (el navegador inclina la upright). Aplica igual a commerce y validation.

Si molesta, la opción es añadir el eje `slnt` a la URL (`opsz,slnt,wght@8..144,-10..0,100..900`), que Chrome y Safari sí usan para `italic` cuando el variable font no tiene `ital`. Cuesta algunos KB más de fuente. **No se cambió** para no dejar el backoffice inconsistente con las otras dos apps; si se hace, hay que hacerlo en las tres.

Ver [[project_cp_app_layout]], [[project_cp_auth_layout]], [[feedback_angular_gotchas]].
