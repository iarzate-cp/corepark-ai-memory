---
name: gotchas-de-angular-y-tooling-descubiertos-en-la-migraci-n-de-layouts
description: "proyección de contenido y @if, selectores de atributo que no fallan, pnpm minimumReleaseAge, publish:lib, orden de styles en angular.json"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-27T03:54:14.527Z
---

Encontrados el 2026-08-26 migrando layouts. Todos costaron tiempo de depuración.

## 1. La proyección de contenido NO atraviesa `@if`

```html
<cp-app-layout>
  @if (isDesktop()) {
    <app-nav cpAppNav />   <!-- ❌ acaba en el slot POR DEFECTO -->
  }
</cp-app-layout>
```

El bloque `@if` se compila como un contenedor sin atributos, así que Angular lo asigna al slot por defecto y todo lo de dentro se renderiza ahí. El atributo del slot tiene que ir en un **nodo declarado estáticamente**.

**Síntoma:** "no veo el layout" — el aside sale vacío y el contenido aparece en `main`.

**Fix:** quitar el `@if` y gestionar la visibilidad por CSS, o envolver en un elemento estático que lleve el atributo.

## 2. Los selectores de atributo no fallan si no importas la directiva

`cpInput`, `cpButton`, `cpNavAction` son selectores de atributo: si olvidas importarlos, Angular **no da error**, simplemente los ignora. (Un selector de elemento como `cp-form-field` sí daría NG8001.)

**Cómo verificar:** buscar el campo `dependencies:[...]` del componente compilado en el bundle, o comprobar que las clases (`.cp-btn`, `.cp-nav-action`) están en el `styles.css` final.

## 3. pnpm 11.x revienta con GitHub Packages

```
pnpm: Invalid time value  at detectMinReleaseAgeViolation
```

GitHub Packages no devuelve el campo `time` que espera el chequeo de `minimumReleaseAge`. Afecta a **frontend-validation**, que lo tiene a `1 day` en `pnpm-workspace.yaml`.

**Funciona:** `pnpm add @corepark/corepark-ui@X.Y.Z --config.minimumReleaseAge=0`
**NO funciona:** añadir la versión a `minimumReleaseAgeExclude` — el crash ocurre antes de consultar la exclusión.

Cualquiera que clone y haga `pnpm install` en frío se topa con esto.

## 4. `publish:lib` necesitaba `./`

`npm publish dist/corepark-ui` sin `./` se interpreta como shorthand de repo git y falla con `Permission denied (publickey)`. Corregido a `pnpm publish ./dist/corepark-ui --no-git-checks` (pnpm se niega a publicar con working tree sucio sin ese flag).

## 5. Orden de `styles` en angular.json

`corepark-ui/styles.scss` debe ir **PRIMERO** en el array, para que los overrides de la app ganen sobre el `:root` del DS. Corregido en commerce (estaba al final). Validation ya lo tenía bien.

**Al editar `angular.json`:** hacerlo con Edit quirúrgico, no re-serializando con `JSON.stringify` — el repo usa tabs y un `stringify(a, null, 2)` reformatea las 300 líneas.

## 6. Recompilar y sincronizar tras cambios de API

`pnpm run build && pnpm run sync:<app>` después de **cualquier** cambio de tipos, no solo de CSS. Si no, el consumidor compila contra los `.d.ts` viejos (p. ej. `TS2741: Property 'route' is missing`).

Ver [[feedback_package_manager]] y [[project_migration_status]].
