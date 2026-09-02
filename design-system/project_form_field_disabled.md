---
name: cp-form-field-estado-disabled-en-escala-de-grises-y-el-bug-del-errorfor
description: el disabled del form-field pasó de opacity a grayscale con doble trigger; y por qué form.disable() hacía aparecer errores falsos
metadata: 
  node_type: memory
  type: project
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-27T22:45:39.856Z
---

Dos cosas que salieron juntas el **2026-08-27**, ambas alrededor de `form.disable()`.

## El bug: deshabilitar un form pintaba errores falsos

Síntoma: al enviar, el form se deshabilitaba y **aparecían mensajes de error** en campos válidos.

Causa, confirmada en `@angular/forms` (`forms.mjs`):

```
1239:    this.status = DISABLED;
1240:    this.errors = null;
1060:  get valid()   { return this.status === VALID; }
1063:  get invalid() { return this.status === INVALID; }
```

Con status `DISABLED`, **`valid` e `invalid` son ambos `false`** y `errors` queda a `null`. El helper hacía:

```ts
if (control.valid) return ''                          // false → no corta
if (control.hasError('required')) return ...required  // false → errors limpiado
return FIELD_ERRORS[kind].invalid                     // ← cae aquí
```

**Arreglo: preguntar en positivo, `if (!control.invalid) return ''`.** Cubre VALID, DISABLED y PENDING de una, y es la semántica que tenía el template de Material (`@if (control.invalid)`), que por eso nunca mostró el falso error.

Estaba en 4 helpers del backoffice **y en `commerce/sign-in`**, que es de donde se copió el patrón. Arreglado en ambos. Verificado ejecutando los dos helpers contra `FormControl` reales en los 5 estados; el único que cambia es `válido + form.disable()`: antes `"Please enter a valid email address"`, ahora `""`.

## El estado disabled del `cp-form-field`

Era `opacity: 0.5` a todo el contenedor. Ahora:

```scss
.cp-field-container.is-disabled,
.cp-field-container:has(:disabled) {
  cursor: not-allowed;
  filter: grayscale(1);
  .cp-label { color: var(--cp-ui-color-text-300); }
  fieldset.cp-notch-outline { border-color: var(--cp-ui-color-text-100); }
}
```

- **`filter` es lo que alcanza el contenido proyectado.** Aplica a todo el subárbol renderizado sin importar la encapsulación, así que un icono de prefijo o el ojo de contraseña del consumidor también se desaturan. Ningún selector del SCSS del form-field puede tocarlos: **el contenido proyectado lleva el atributo de encapsulación del consumidor, no el del componente.**
- **Doble trigger, y hacía falta.** `.is-disabled` solo aparece si el consumidor pone `[disabled]` en `cp-form-field`. Cuando un reactive form llama a `form.disable()`, se pone el `disabled` **nativo** en el input y el input del componente no se toca — así que antes un form enviándose dejaba los campos con pinta de editables. `:has(:disabled)` cubre ese camino.
- La regla de hover tuvo que moverse al contenedor por lo mismo: `:not(.is-disabled)` en el fieldset se perdía el camino de reactive forms.
- **Sin fondo gris**, a propósito: la etiqueta flotante se sienta a `translateY(-50%)` sobre el borde, o sea medio fuera del contenedor, y un fondo dejaría la mitad de arriba del texto sobre el color de la página.

## De paso: `.cp-field-input` tenía mal la casa

La clase que aplica `cpInput` se estilizaba dentro de **`select-component.scss`** y funcionaba en los form-fields solo porque esa hoja es global. Movida a `lib/directives/input/input-directive.scss` y registrada en `styles.scss`. Tiene que ser **global** por lo mismo de la encapsulación del contenido proyectado. Se le añadió `-webkit-text-fill-color` en `:disabled`, porque WebKit pinta su propio gris sobre `color` e ignora la cascada.

Ver [[project_corepark_ui_components]] y [[project_migration_status]].
