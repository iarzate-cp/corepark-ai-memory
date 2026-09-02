---
name: project_calver_apps
description: las apps usan CalVer YYYY.M.N con pnpm run release; la librería se queda en SemVer; por qué y qué lo rompe
metadata: 
  node_type: memory
  type: project
  originSessionId: aea67aa5-d32a-4d47-b014-93e055e5f1b9
  modified: 2026-08-31T22:05:44.970Z
---

Decisión de Israel del **2026-08-28**: las **apps** pasan a CalVer, la **librería no**.

## El esquema — CORREGIDO el 2026-08-31

`YYYY.M.D` — año, mes y **día real del build**. `2026.8.31`, `2026.9.1`, `2027.1.9`.

> **El contador fue un error, y lo detectó Israel.** El esquema original tenía el tercer campo como contador dentro del mes (`2026.8.0` → `2026.8.1`). El primer uso real lo rompió: se selló `2026.8.1` el **31** de agosto, y eso **se lee como el 1 de agosto**. Una cadena con forma de fecha que no es la fecha es peor que un contador opaco, porque la gente se la cree — y todo el sentido del esquema es que alguien la lea y confíe en ella. Es exactamente el fallo que el propio razonamiento de la decisión advertía.
>
> Detalle que ayuda a ver el problema: `2026.8.0` **no engaña**, porque no existe el día 0. Son `.1` a `.28` los que mienten con cara de verdad.

**Dos releases el mismo día comparten versión, y es deliberado.** El día es lo que la gente cita; el **SHA corto** es lo que distingue builds — un contador no daba ninguna de las dos cosas. Por eso el SHA pasa de "mejora" a parte del diseño.

`scripts/release-version.mjs` + `pnpm run release` en cada repo. Replace textual sobre el `package.json` (no re-serializa) para que el diff sea una línea. **Ahora es idempotente**: correrlo dos veces el mismo día imprime *"already today's date, nothing to stamp"* en vez de incrementar.

Las tres apps arrancaron en `2026.8.0` el 2026-08-28, viniendo de `1.26.0` (backoffice), `3.4.2` (commerce) y `1.8.2` (validation). **Todas suben, ninguna baja**: `2026 > 3`, así que ninguna herramienta lo lee como downgrade.

**Sin ceros a la izquierda**: `2026.08.31` no es SemVer válido y pnpm lo rechaza. `2026.8.31` sí lo es (verificado), igual que un cuarto campo **no** lo sería (`2026.8.31.1` inválido) — por eso el día no puede convivir con un contador.

**Estado al cierre del 2026-08-31:** commerce y backoffice en `2026.8.31`, ambos con el script por día. **Validation se quedó atrás**: script del contador y `2026.8.0`. El script es idempotente — correrlo dos veces el mismo día no incrementa, así que se puede correr al cortar sin miedo.

## Por qué CalVer aquí y SemVer allí

SemVer existe para decirle a **quien depende de ti** qué puede romperse. Nadie declara `frontend-backoffice: ^1.26.0`, así que en una app desplegada esos tres campos codifican información que nadie lee. La pregunta real es "de cuándo es este build" — soporte la lee en la pantalla de sign-in.

`corepark-ui` **sí** tiene cuatro consumidores resolviendo rangos, así que se queda en SemVer. Que el esquema de las apps no se contagie a la librería.

Dato que salió al revisarlo: en npm **`^0.0.28` equivale a `0.0.28` exacto** — el caret sobre `0.0.x` no permite ningún cambio. Por eso los cuatro consumidores están pineados de hecho aunque dos usen caret, y por eso el `0.0.x` del DS no comunica compatibilidad alguna. Ver [[project_pending_work]].

## La condición que lo hace o lo rompe

**No hay CI que construya** en ninguno de los cuatro repos — solo `slack-pr-approved.yml`. Así que el sello lo pone quien corre `pnpm run release`.

Una fecha escrita a mano miente la primera vez que alguien olvida actualizarla, y una versión que **afirma** una fecha falsa es peor que un contador sin significado, porque la gente confía en ella para triage. Si esto empieza a rotar, la salida es automatizar el bump, no volver a SemVer.

Pendiente que lo cerraría: estampar el SHA corto junto a la versión (`2026.8.0 · a5be2fe`). La fecha es lo que la gente cita; el SHA es lo que sirve para triage.

## Dónde se muestra

`footerText` de `cp-auth-layout`, en las tres. Backoffice: `Version 2026.8.0`. Commerce: `Corepark Parking Management · 2026.8.0`. Validation: `Corepark Partner · 2026.8.0`.

**valet-web quedó fuera** (no muestra versión en ninguna parte y usa su `auth-layout` propio, no el del DS).

## `import { version }`, nunca `import pkg`

`import pkg from '@package'` **empaqueta el `package.json` entero** en el bundle del navegador — `devDependencies`, `scripts`, todo. Con `import { version } from '@package'` esbuild poda el resto. Verificado en los tres bundles buscando un marcador único del archivo.

El BO necesita el alias `version as appVersion` porque sus dos auth layouts ya tienen un campo `readonly version`.

Trampa al verificar: buscar `devDependencies` en el bundle da **falso positivo** — hay librerías de terceros que embeben su propio `package.json`. Hay que buscar una cadena única del archivo de la app.
