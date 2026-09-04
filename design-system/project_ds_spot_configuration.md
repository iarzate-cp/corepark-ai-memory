---
name: project_ds_spot_configuration
description: "ds-spot-configuration: rebuild del acordeón de garajes — LAYOUT_SHAPE mata el enum SpotConfig, los nueve defectos del clásico, y el código muerto que se dejó a propósito"
metadata:
  node_type: memory
  type: project
  modified: 2026-09-04
---

Hecho el **2026-09-04** en `feature/design-toggle`, commit `0a09f54f`, ya en `feature/staging`. Ruta duplicada con `newDesignGuard`; el módulo clásico intacto.

**Sin verificar en pantalla.** Israel lo encuadró como «un módulo que no me encanta cómo lo desarrollé».

## La forma: acordeón, no tabla

Se la propuse como tabla + diálogo (el patrón de rates y lodging) y **eligió acordeón reconstruido**: la distribución de un garaje es un formulario, y va en la página, no tras un modal. Lo que cambia es todo lo de debajo.

**1 313 líneas** contra las ~1 772 vivas del clásico, haciendo más.

| Clásico | Nuevo |
|---|---|
| `spot-config-toggle-collapse` (392) + `spot-config-new-garage` (267) | `ds-garage-card` — `garage` a `null` es lo que lo hace un alta |
| `just-rows-form` + `just-spots-form` + `letters-to-numbers-form` + `numbers-to-letters-form` | `ds-layout-fields` |
| 4 `FormArray` hermanos, 3 siempre vacíos, 5 métodos para limpiarlos | 1 `FormArray rows` |
| Enum `SpotConfig` + 2 mapeos con nombre + 2 inline | Solo `LayoutType` |

## La pieza clave: `LAYOUT_SHAPE`

`@utils/spot-config-layouts`. Cada `LayoutType` declara de qué está hecha una fila (`rowKind`, `spotKind`), y de ahí salen qué campos se pintan, qué validadores llevan y qué va a `null` en el payload. **Añadir un layout es una fila en esa tabla, no un quinto componente.**

El enum `SpotConfig` tenía los mismos cuatro valores que `LayoutType` en kebab-case, así que cada lectura y cada escritura convertía entre ambos. Hablar un solo vocabulario **elimina la conversión en vez de arreglarla**.

## ⚠️ Los dos mapeos invertidos siguen vivos en el clásico

`core/utils/spot-config-utils.ts`, líneas 65-85: `spotConfigByLayoutType` y `layoutTypeIntoSpotConfig` tienen los dos casos de rows-and-spots **al revés**.

Sobreviven solo porque **nadie los llama**: los dos componentes reimplementan la conversión inline y bien, por duplicado. `layoutTypeIntoSpotConfig` se importa en `spot-config-toggle-collapse:9` y nunca se invoca.

**El día que alguien «limpie» usándolos, intercambia los dos layouts en silencio.** Los validadores confirman cuál es la buena: `lettersToNumbers` pone `justLetters` en `row` y `justNumbers` en `from`/`to`, o sea filas alfabéticas y cajones numéricos = `AlphabeticalRowsNumericalSpots`.

Encima los nombres se contradicen entre sí: `spotConfigByLayoutType` recibe un `LayoutType` y `layoutTypeIntoSpotConfig` recibe un `SpotConfig`. Cada nombre dice lo contrario de lo que hace.

## Los nueve defectos del clásico

Además de los mapeos:

1. **La X de Numbers→Letters no hace nada**: `numbers-to-letters-form:36` borra de `lettersToNumbers`, copiado del hermano. Ese array está vacío en ese layout, así que `removeAt` es un no-op silencioso. Es la consecuencia de que los dos componentes sean el mismo fichero dos veces.
2. **Just Spots no deja añadir ni quitar rangos** en ninguna de las dos plantillas, aunque las dos llevan una rama inalcanzable de `addRow()` para él.
3. **`from > to` no se valida.** El template pinta ramas `invalidFrom`/`invalidTo` que ningún validador puede poner: se guarda el rango 50→1.
4. **Tras crear, el estado se reconstruye buscando por nombre** (`new-garage:227`) y **añadiendo sobre `prev`**, descartando el GET recién hecho. Nombre duplicado → coge el primero; el backend normaliza el nombre → el garaje nuevo no aparece nunca.
5. **`updating` tiene dos escritores**: un `effect` lo deriva de `isExpanded` y además se asigna a mano en cinco sitios, así que el effect pisa al código imperativo en cada expansión. Es un `computed` disfrazado.
6. **`#setForm()` dentro de un `effect` puede resetear el formulario a media edición.** Borrar *otro* garaje refresca la lista, llegan objetos nuevos, el input de la fila que editas cambia de referencia y el effect llama a `form.disable()`. Deducido leyendo, no reproducido.
7. **`console.error` y página en blanco** si falla el GET (`spot-config-page:57`).
8. **El primer garaje no se puede cancelar**: la página siembra el borrador y el botón de descartar está tras `@if (parkingLayoutAreas().length > 0)`.
9. **El layout se elige dos veces**: tarjetas con imágenes + `mat-menu` para la cuarta opción, y justo después un `mat-select` con las cuatro.

## Código muerto — se deja a propósito

Israel: nada por ahora, se limpia cuando se retire el clásico. **~1 631 líneas, casi tanto como lo vivo:**

| Qué | Líneas |
|---|---|
| `pages/main/settings/spot-configuration/` (módulo anterior entero) | 1 413 |
| `shared/components/rows-and-spots-form/` (su `removeRow` tiene el cuerpo comentado) | 195 |
| `core/services/settings/spot-config/` (servicio duplicado + spec) | 23 |

Más, dentro de lo vivo: **4 de las 5 funciones de `spot-config-utils.ts`** no se llaman, y `getParkingLayoutAreaById` del servicio tampoco. `numberVerifier` es una función cuyas dos ramas devuelven lo mismo.

## Decisiones que no conviene revisitar

- **Las filas de «Solo filas» siguen aceptando cualquier cosa.** El campo se etiqueta «Alphabetic row» pero el clásico solo le pone `required`, así que puede haber ubicaciones con filas tipo `A1`. Ponerle `justLetters` las dejaría imposibles de guardar en cuanto alguien las abriera. Marcado como `'free'` en `LAYOUT_SHAPE`.
- **El tipo de layout sigue bloqueado al editar**, como en el clásico: cambiarlo invalidaría todos los cajones ya registrados.
- **`spotRangeValidator` compara letras por longitud primero**, porque las etiquetas van A…Z y luego AA, y `'AA' < 'B'` lexicográfico es falso ahí. Los números, numéricamente.

Ver [[project_material_to_corepark_ui]], [[project_ds_rates]] y [[project_garage_spot_uniqueness]].
