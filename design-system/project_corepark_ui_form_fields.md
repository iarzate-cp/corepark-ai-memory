---
name: project_corepark_ui_form_fields
description: "La familia de campos de corepark-ui: el CVA que se añadió, cp-time-field y cp-date-range-field, y los dos cambios de comportamiento de cp-form-field que afectan a las tres apps"
metadata:
  node_type: memory
  type: project
  modified: 2026-09-03
---

Todo esto es del **2026-09-03**, en `develop` de `~/Dev/design-system`, **sin publicar**. 394 tests pasan (eran 311 al empezar).

## ControlValueAccessor donde no lo había

`cp-checkbox`, `cp-switch` y `cp-radio-group` llevaban solo un `model()`, así que **un `formControlName` encima no enlazaba con nada** y el control se quedaba con su valor inicial en silencio. Eso los descartaba de las formas de las tres apps, que son reactivas.

El arreglo es **aditivo** (`d305a36`): el `model()` sigue siendo la fuente de verdad —`writeValue` lo escribe, la interacción lo actualiza— así que `[(checked)]` y `[(value)]` siguen funcionando. Commerce los usa en 26 sitios y no se enteró.

**El detalle que no era obvio:** `disabled` es un `input()`, así que un formulario llamando a `setDisabledState` no puede escribirlo. Va un `#cvaDisabled` **al lado** y un `isDisabled()` que los combina, que es lo único que leen el host y el toggle. Misma forma que ya tenía `cp-select`.

`cp-date-range-picker` y `cp-time-picker` **siguen sin CVA**: se usan a través de sus campos nuevos, no directamente.

## Dos campos nuevos, y ningún picker nuevo

Los dos paneles existían y **no se podían usar en un formulario**: `value` de entrada, `apply` de salida, sin campo, sin trigger y sin CVA. Solo se manejaban de forma imperativa, y por eso las tres apps seguían tirando de `nxt-pick-datetime`.

**`cp-time-field`** (`a856db0`) — dispara `cp-time-picker`. Valor **`'HH:mm'`**, el string que el panel ya habla, no un `Date`: un `Date` arrastra día y zona que una hora de reloj no tiene, y es lo que convierte «18:00» en ayer por la tarde en otro sitio. Tolera `'HH:mm:ss'` al entrar y lo trunca (es lo que mandan las APIs); **nunca lo emite** — quien necesite segundos añade `':00'`. Lo impersable pasa a `null`, para que el control no guarde algo que el panel renderizaría como 09:00.

**`cp-date-range-field`** (`a4b9dd3`) — dispara `cp-date-range-picker`. Valor `DateRange` de `DateTime` de Luxon. **No hizo falta crear picker**: ese panel ya hace fecha **y** hora en los dos extremos, con AM/PM y default 12:00 AM → 11:59 PM, que es exactamente una ventana de validez. Solo faltaba el campo. Un `DateTime` inválido se rechaza a `null`: Luxon no lanza en un parse malo, devuelve un objeto cuyo `toFormat` es el string «Invalid DateTime».

**Los dos copian el montaje de overlay de `cp-select`**: conectado al campo, backdrop transparente, `reposition()` en scroll.

**Sigue faltando** un picker de **fecha única** — 15 ficheros del BO lo piden (`mat-datepicker`). Es el único hueco de la lib que queda, y es componente público nuevo.

## Dos cambios de comportamiento de `cp-form-field`

Afectan a **las 51 vinculaciones `[error]`** de las tres apps. Son arreglos, no efectos colaterales, pero hay que saberlo.

### El error espera al blur (`b46f0c3`)

`cp-form-field` pintaba el string de `error` en cuanto lo tenía. Un control `required` es inválido desde el primer render, así que **el formulario regañaba antes de que nadie escribiera**.

Material tenía la regla dentro del componente y no nos dimos cuenta de que dependíamos de ella: `mat-error` solo se renderiza cuando el `errorState` del form field es true, y el `ErrorStateMatcher` por defecto es

```
control.invalid && (control.touched || form.submitted)
```

O sea que el `@if (control.invalid)` de las plantillas clásicas **parecía el filtro y era decoración**.

Ahora hay `errorOn: 'blur' | 'always'`, con `'blur'` por defecto. **Va por el `focusout` del propio campo, no por `control.touched`**: `touched` es un getter plano, así que un `computed` que lo lea nunca recomputa; el blur es el mismo instante y encima funciona para los wrappers cuyo control vive por fuera (`cp-select`, `cp-time-field`, `cp-date-range-field`), donde no hay `NgControl` proyectado.

El contorno y la etiqueta flotante siguen el mismo gate, para que el campo no se ponga rojo con el mensaje oculto.

`'always'` para lo que no viene de la validación del campo — un rechazo del servidor — donde esperar un blur que puede no llegar lo esconde.

**Excepción que no cubre:** un error de grupo (`FormArray` o `FormGroup`) no cuelga de ningún `cp-form-field`, así que ahí el `touched` se comprueba a mano. Pasa en el horario de las tarifas y en los importes de tipping.

### El label flota si el wrapper lo dice (`9e6f6e7`)

`cp-form-field` deduce si tiene valor de tres sitios: el valor del input al primer render, el `valueChanges` de su control, y el evento `input`. **Un wrapper no dispara ninguno** — su input interno es `readonly` y se rellena con `[value]`, así que no hay `formControlName` al que suscribirse y una escritura de propiedad no emite `input`.

Resultado: `cp-time-field` y `cp-date-range-field` **pintaban el valor encima del label**, los dos en el mismo sitio.

Ahora hay un input `hasValue: boolean | null`. `null` (default) deja al campo deducirlo, que es lo que quiere un input proyectado normal.

**`cp-select` tenía el mismo problema** y lo esquivaba **disparando un evento `input` sintético** contra el form field. Ya lo declara igual y el evento falso se fue. Sigue escribiendo el texto a mano en el input, porque ese elemento lleva la búsqueda mientras el panel está abierto.

## `cp-menu` salió del inline al overlay del CDK (`b2bb241`)

Pintado inline heredaba el contexto de apilamiento donde se declaraba —dentro del rail, el aside— y cualquier ancestro con `filter`, `transform` o `backdrop-filter` se volvía bloque contenedor de su `position: fixed` y lo recortaba. Eso lo descartaba de tablas y de capas compositadas, y **bloqueaba las container queries** del área de contenido.

Gratis con el cambio: el CDK voltea el panel cuando no cabe (se fueron los umbrales a mano de `#autoPosition`) y `reposition()` lo mantiene pegado al trigger al hacer scroll.

**API de consumidor idéntica**: `<cp-menu #m>` + `[cpMenuTrigger]="m"`. Verificadas las 101 referencias de las cuatro bases. `MenuCoords` salió del barril (nadie lo importaba) y el knob `--cp-ui-menu-z-index` también: quien necesite subir el menú sube `.cdk-overlay-container` y ordena todos los overlays a la vez.

**Primeros tests del componente, que no tenía ninguno** — y es donde vive el bug del `disabled` de `cp-menu-item`.

## Sin verificar en pantalla

Nada de esto se ha visto. Son **tres componentes nuevos** y **dos cambios de comportamiento** que llegan a 51 campos de tres apps. Ver [[project_pending_work]].
