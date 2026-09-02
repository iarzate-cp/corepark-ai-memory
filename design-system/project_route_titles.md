---
name: project_route_titles
description: "títulos de página desde la ruta con TitleStrategy en las tres apps; formato de pestaña, data.meta/subheading, y la trampa de herencia de data"
metadata: 
  node_type: memory
  type: project
  originSessionId: aea67aa5-d32a-4d47-b014-93e055e5f1b9
  modified: 2026-08-29T00:14:02.332Z
---

Montado el **2026-08-28** en backoffice, commerce y validation. Sustituye tres copias en la sombra del árbol de rutas.

## El patrón

La ruta declara su nombre; una `TitleStrategy` propia lo traduce (si aplica), lo publica como señal y pinta la pestaña.

```ts
{ path: 'rates', title: 'NAVIGATION.SETTINGS.RATES', data: { meta: 'breadcrumb' }, loadComponent: … }
```

- **BO**: `TranslatedTitleStrategy` en `@routing/page-title`. Los `title` son **claves i18n**.
- **commerce / validation**: `PageTitleStrategy`. **No tienen ngx-translate**, así que el `title` es texto de display.

**Registro con `useExisting`, nunca `useClass`:**

```ts
{ provide: TitleStrategy, useExisting: TranslatedTitleStrategy }
```

Con `useClass` se construye una **segunda instancia**: el router mueve una y el header lee la otra. No da error, simplemente el título nunca cambia.

## Formato de pestaña — decidido por Israel

```
Valet App Config | Settings | CorePark - CRM
```

`CorePark - BackOffice` · `CorePark - CRM` · `CorePark - Partner`. Separador ` | `.

- **`buildTitle()` no sirve** para esto: solo devuelve el título más profundo. Correcto para el `<h1>`, pero pierde la sección. La cadena se construye recorriendo la rama con `getResolvedTitleForRoute()` en cada nodo, **recursivo y leaf-first**.
- **Dedupe de segmentos consecutivos iguales**: `/locations` declara el mismo nombre en la sección y en su landing; sin eso la pestaña dice "Locations | Locations".
- **El `<h1>` no lleva la cadena**, solo el nombre de la página.
- La pestaña se pinta con un **`effect`**, no dentro de `updateTitle`: en arranque en frío el JSON de i18n puede ir en vuelo y `instant()` devolvería la clave cruda sin nada que la corrija.

## La trampa de la herencia de `data`

**Con `paramsInheritanceStrategy` por defecto, un hijo con path NO vacío no hereda el `data` del padre.** Leer del snapshot más profundo pierde lo que declara un ancestro.

Mordió de verdad: en commerce `/aggregators/connections` redirige siempre a `ocra`, que no declara subtítulo → el subtítulo del padre **nunca se renderizaba**, en silencio.

Solución en commerce y validation: `closest(node, key)` recorre la rama y toma **el más cercano declarado desde la hoja hacia arriba**. Deja de depender de la estrategia de herencia.

## Qué lleva `data`

| Clave | App | Para qué |
|---|---|---|
| `meta` | BO | `'breadcrumb' \| 'location' \| 'none'` — qué va bajo el título. Default `'location'` |
| `subheading` | commerce, validation | Strapline de la página |
| `ownHeader` | commerce | La página monta su propio header (necesario si lleva acciones) |

## Lo que se borró

- BO: `HEADLINING_TITLES` (29 entradas), `GLOBAL_MODULES`, `STANDALONE_MODULES`, `wrapper.titles.ts` y el pipe de `router.events` del wrapper. **31 rutas** con `title`, 6 con `data.meta`.
- commerce: **27 headers a mano** (`.page-header` + su SCSS en 27 archivos). El header lo pinta `DsAppLayout` una sola vez, y solo `@if (heading())` — una ruta sin título no recibe header, y ese `@if` **es el opt-out**.
- validation: 3 rutas, header en el shell. Los otros shells (kiosk, valet-dashboard, monitores) no declaran título y quedan desnudos.

## Basura que salió al desmontar los mapas

- `/transfer-ticket` era una entrada **muerta**: no existe esa ruta (`TransferTicketPage` vive en `settings/stations`).
- `'Stations'` era **texto crudo, no una clave** — funcionaba porque `instant()` devuelve la entrada cuando no la encuentra. Existía `NAVIGATION.SETTINGS.STATIONS`.
- **Cuatro páginas del BO no tenían título** y renderizaban un `<h1>` vacío con sus 32px de margen: `request-points`, `pin-codes`, `reports/scheduled`, `payments/credentials`.
- `Layout.Backoffice.Header.Title` en los 3 JSON: nadie lo referencia.
- `/payments` estaba en `GLOBAL_MODULES` pero esa página nunca usó el wrapper principal.

Ver [[project_cp_page_header]] y [[project_backoffice_wrapper]].
