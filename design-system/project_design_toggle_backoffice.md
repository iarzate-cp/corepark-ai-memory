---
name: project_design_toggle_backoffice
description: "backoffice: toggle legacy/nuevo gateado por operador; los seis fallos visuales que solo aparecieron comparando DEV con producción"
metadata: 
  node_type: memory
  type: project
  originSessionId: 02d8be60-b4d7-4d73-99ae-4bda6564aa7f
  modified: 2026-09-01T23:05:03.752Z
---

Construido el **2026-08-31** en `~/Dev/frontend-backoffice`. Misma idea que [[project_design_toggle_commerce]] pero con dos requisitos distintos de Israel: **solo para operadores 1 y 2**, y **el login también cambia** si el usuario activa el nuevo.

## Ramas

- **`feature/design-toggle`** (`ea4cc07e`) ← nació **de `main`**, que había avanzado 2 commits, y trajo el diseño mergeando `feature/app-layout`.
- **`feature/staging`** ← mergeada y empujada. Staging es **sumidero**, ver [[feedback_branching_staging]].
- Al cierre del **2026-09-01**: app en `2026.9.1`, lib en **`0.0.30` del registro**, ambas ramas empujadas (`8bacf819` y el merge `13558797`). **Ninguna mergeada a `main`.**
- Encima del toggle vive la reconstrucción de seis módulos sobre la lib — ver [[project_bo_module_migration]].

## El gate por operador — hay un patrón de la casa, no inventar otro

`core/utils/design-access.ts` es **copia de la forma de `canAccessActivityByRateClass`**, que ya existía:

```ts
export const canAccessNewDesign = (operatorCompanyId: number | null): boolean => {
	if (!environment.production) return true
	if (operatorCompanyId === null) return false
	return ALLOWED_OPERATORS.includes(operatorCompanyId)   // CorePark(1), SecureParking(2)
}
```

**Fuera de producción está abierto a todos** — lo preguntó Israel y el patrón de la casa ya lo respondía. Coste que hay que tener presente: **el camino gateado pasa a ser el que nadie ejercita antes de publicar**; comprobar que el toggle *no* aparece solo se puede hacer contra prod o cambiando el flag a mano.

Usar `ProductionOperators.CorePark` / `.SecureParking`, nunca los literales 1 y 2.

## Dos banderas, porque el gate y el login viven en momentos distintos

`operatorCompanyId` sale del JWT que `operatorDataFactory` decodifica **sincrónicamente** al arrancar, así que post-login el gate es inmediato y sin destello. Pero **en el login no hay token**.

- `isNew` = preferencia **Y** elegibilidad → la app.
- `prefersNew` = solo preferencia → las pantallas de auth.

Un operador no elegible con `new` guardado ve **login nuevo y app legacy**, y **se conserva la preferencia** (decisión de Israel: un elegible que vuelve al mismo dispositivo no tiene que reactivarla).

`DesignState.stamp()` escribe el diseño **efectivo**, no la preferencia cruda — si no, un no elegible tendría `data-design="new"` con el shell legacy. Lo estampa el initializer (sincrónico, pre-render) **y** un `effect`, porque la elegibilidad cambia al iniciar sesión sin recargar.

⚠️ **`designProvider` debe registrarse DESPUÉS de `operatorDataFactory`** en `app.config.ts`.

## Los shells: componentes, no `canMatch`

`design-shell` y `design-auth-shell` eligen con un `@if`. **Distinto de commerce a propósito**: la ruta raíz del BO lleva **46 rutas hijas inline**, y duplicar la entrada para `canMatch` habría obligado a duplicarlas o extraer un const de ~300 líneas. En commerce ya eran un const, así que allí fueron 6 líneas.

**Los dos llevan `:host { display: contents }`** y es obligatorio: `main-layout` es `display: grid; height: 100svh; overflow: hidden`, y meterlo bajo un host `inline` hace crecer el documento y aparece una barra de scroll que producción no tiene.

## El header de página: NO ramificar el wrapper

Un solo `wrapper` sirve a **93 páginas**, y su look legacy se resolvió **solo con estilos**: knobs de `cp-page-header` bajo `[data-design='legacy']` más el `padding: 4rem` que el shell viejo no aporta.

La razón de no ramificar: las dos ramas necesitarían `<ng-content select="[actions]">` y `[content]`, y **duplicar un selector entre ramas rompe la proyección** — solo el primero recibe. (El `@if` en sí no es el problema; ver la corrección en [[feedback_angular_gotchas]] 1b.)

Casi todo coincidía ya, porque **`cp-page-header` se extrajo de este mismo wrapper**: su `--meta-gap` es el `0.25rem` del `h1` viejo y su `--gap` los `2rem` del `header` viejo.

## Auth legacy con fidelidad total

Israel eligió fidelidad completa: las plantillas de Material vuelven en las 3 páginas detrás de un `@if`, con `MatFormField/Input/Icon/Button` y `ActionerRaised` de vuelta en los imports.

- **No** se restauró `sign-in-material.module.ts`: solo lo consumía `sign-in.module.ts` y ambos estaban borrados. Los módulos van directos al array standalone.
- Dos adaptaciones en `config-new-password`: `hide`/`hideConfirm` son signals ahora (el markup viejo hacía `hide = !hide`), y el literal `20` es `passwordMaxLength`. Reponer booleans mutables junto a los signals daría dos fuentes de verdad al mismo toggle.

## LOS SEIS FALLOS QUE SOLO APARECIERON COMPARANDO CAPTURAS

Esto es lo más valioso de la sesión. Israel ponía DEV y PROD lado a lado y yo buscaba la causa. **Cada uno era una sola línea**, y ninguno lo habrían cazado mis auditorías —el CSS existía y compilaba, simplemente no aplicaba.

1. **Formulario de login sin estilo.** La migración también reescribió `auth-forms.scss`. Faltaban `legend`, `footer` y `.anchor`. Restaurados literales — **sin gating**, porque los selectores de los dos diseños son disjuntos.
2. **Barra de scroll de más.** Los dos `design-shell` sin `display: contents` (arriba).
3. **Fondo claro + botones del header invisibles = UNA causa.** `_normalize.scss` cambió `body` de `--color-grey-500` a `--cp-ui-color-bg-app`. Los textos del header son `white-on-black` por diseño → blanco sobre blanco. Fix: `html[data-design='legacy'] body`.
4. **Logo oscuro sobre el aside negro.** `operator-logo` renderiza `<cp-brand />`, que por defecto es `--cp-ui-color-text-950`.
5. **Tipografía.** Global a Roboto Flex. Ahora `--app-font-family` en `:root`, re-apuntada bajo `[data-design='legacy']` a Lato, leída por el `*` que ya existía. Se queda en `*`: moverlo a `body` rompe los controles de formulario.
6. **Headings más gruesos.** Los knobs de `cp-page-header` puestos en un ancestro.

**Los fallos 4 y 6 son el mismo error, cometido dos veces**: ver la regla de los knobs en [[feedback_design_system_patterns]]. Y en el 4 hicieron falta **dos mecanismos**, porque `cp-brand` expone knob (wordmark, CSS) e input (`isotypeColor`, binding) — arreglar solo uno deja el lockup a medio pintar.

**Método que funcionó y conviene repetir:** capturas DEV/PROD lado a lado. Yo puedo verificar que el CSS existe en el bundle; no puedo ver cómo se ve.

## Sin verificar — por orden de importancia

1. **Nadie ha pulsado el toggle.** El overlay, el `reload()` y todo el **diseño nuevo del BO** están sin ver. Lo depurado es el legacy por defecto.
2. **Un operador NO elegible.** Y con el gate abierto fuera de producción, ya **no es verificable en dev**.
3. **Todo el legacy móvil** (`mobile-nav`, `mobile-container`, `mobile-options`, `mobile-user-data` — ahí metí un botón).
4. `recovery-password` y `config-new-password`: solo se vio sign-in.
5. La rama de marca blanca de `operator-logo` (12 logos de operador).

## Diferencia con producción que queda abierta

**El logo es un 14% más pequeño.** `cp-brand` usa `subtitle-1` (1.875rem) e isotipo 2.25rem; el `corepark-imagotype` de producción usaba `--fs-extra-big` (2rem) e isotipo 3rem. Se arregla con el knob `--cp-ui-brand-word-size` sobre `cp-brand` y el `isotypeHeight` por input — no se hizo.

Ver [[project_pending_work]] y [[project_responsive_shell]].
