---
name: Feature — captura de teléfono y expected departure en el gate de Card on File
description: Pantalla "Unlock your valet pass" pide teléfono y expected departure antes de guardar la tarjeta; configurable por location (oculto/opcional/obligatorio) vía el catálogo custom.cat_screen. 5 repos en feature/staging el 2026-09-08, sin PR a main.
metadata:
  type: project
---
En la pantalla **"Unlock your valet pass"** (el gate de Card on File) el huésped captura **teléfono** y **expected departure** antes de `Save card`. Cada campo se configura por location como oculto, opcional u obligatorio.

Nació del documento `guest-page-unlock-guest-info-suggestion.md` (2026-09-04). El teléfono se entregó primero; el departure se agregó el 2026-09-08 porque estaba en el requerimiento original y se había quedado fuera.

## La decisión que define todo: catálogo, no columnas

El documento proponía 3 columnas nuevas en `company.guest_page_cfg` (§4.1) y como alternativa reusar el modelo de screen config (§4.3). **Se eligió el catálogo**, y fue la decisión correcta:

```
custom.client_app_type   ('guest_page')            ← primer client no-valet_app
  └─ custom.cat_screen   ('unlock_valet_pass')
       └─ custom.cat_screen_field  ('phone', 'expected_departure')
            └─ company.field_configurations  (is_visible + is_required por location)
```

El pago: **agregar el segundo campo fue un solo `INSERT`**. Cero cambios en ms-backoffice-service y cero en Commerce — el endpoint admin es genérico sobre `client_app_key` y Commerce pinta lo que devuelva. Con columnas habrían sido 4 repos.

`cat_screen.client_app_key` tiene FK a `custom.client_app_type`, que solo tenía `valet_app`: hay que registrar la app **antes** de la pantalla. Ese insert faltaba en el primer DDL y lo cachó el reviewer.

## Contrato de zona horaria (lo más fácil de romper)

El huésped captura **fecha y hora**, no solo fecha, así que el backend no sintetiza ninguna hora. El valor viaja como **fecha-hora local sin offset** (`yyyy-MM-ddTHH:mm`) y `parking_location_tz` resuelve el instante.

Es deliberado: la zona del dispositivo del huésped no es la de la location. Alguien reservando desde México para una location en Los Angeles que elige "el 10 a las 11:00" quiere decir 11:00 donde está el carro. Es el inverso exacto de cómo `ticket-info` lee la columna, así que el valor da la vuelta a la misma hora de reloj. Un departure pasado se rechaza con `INVALID-EXPECTED-DEPARTURE`.

## Por qué el hotel room aparece en código que no lo captura

`ms-firebase-service` indexa el mapa de campos **por status** (`MQTicket:445-484`):

- `EDIT_TICKET_INFO` (9) lleva teléfono y NO lleva room ni departure.
- `EDIT_HOTEL_INFO` (15) lleva `hotelRoom` **y** `expectedDeparture`, juntos y sin condición.

Y el mapa va directo a `updateChildrenAsync` (`FirebaseDaoImpl:443`) sin filtrar nulos. En RTDB una llave con valor null **borra el hijo**. Entonces:

- Hacen falta **dos** notificaciones, una por bloque. Un departure bajo el status 9 nunca llegaría al RTDB.
- El `hotelRoom` viaja **con su valor leído de la BD** aunque la guest page nunca lo pida, o guardar un departure borraría el cuarto que capturó el valet. Por eso `TicketEditInfoSnapshotBean` se extendió con los dos campos.

Alternativa que se consideró y no se tomó: agregar `expectedDeparture` al mapa de `EDIT_TICKET_INFO`. Sería una sola notificación y el room desaparecería de aquí, pero cambia un contrato que también consumen la valet app y el portal de validaciones.

## Se recarga ticket-info al activarse la tarjeta

`ticket.component` reacciona a `TicketInfoState.guestInfoRefreshRequests` (mismo patrón que `paymentRefreshRequests`) y relee `ticket-info`.

Sin eso hay bug real: la página que el gate desbloquea decide qué pedir leyendo los campos del ticket — `ticket.component:109` pide teléfono si `hasSurvey && !guest.phoneNumber` — así que al huésped le volvían a pedir el teléfono segundos después de escribirlo. El listener de Firebase no lo salva: `FirebaseState` **lee** `TicketInfoState`, no lo rehidrata.

Es lectura simple, no polling: el PATCH se commitea antes de llamar a Stripe, no hay carrera de webhook como en el pago.

## Ramas (2026-09-08, todas en feature/staging, ninguna en main)

| Repo | Rama | Feature | Merge a staging |
|---|---|---|---|
| frontend-guest-page | `feature/unlock-phone-field` | `bde18d22` | `c0b06c10` |
| frontend-commerce | `feature/commerce-guest-page-screen-config` | `3ef4695` | `6756873` |
| ms-valet-service | `feature/unlock-guest-info` | `716d1d12` | `d183ed52` |
| ms-backoffice-service | `feature/guest-page-screen-config` | `e14b044` | `32078e3` |
| ms-gateway-service | `feature/guest-page-screen-config-permit` | `74e675e` | `4f039a9` |

DDLs (los dos corridos en dev): `DDL_2026_09_07_GUEST_PAGE_SCREEN_CONFIG` y `DDL_2026_09_08_GUEST_PAGE_UNLOCK_DEPARTURE`. `~/Dev/Back-End/ddl` **no es repo git**, así que esos scripts viven solo en disco.

**Orden sugerido de PRs a main:** gateway y backoffice pueden ir solos (abren una ruta que nadie llama), luego valet-service, y **la guest page al final** — su `catchError` convierte el 404 del endpoint nuevo en "campo oculto" en silencio, así que si entra primero no falla, simplemente no muestra nada.

## Pendientes

- **`requireCardOnFile` nunca se implementó** en ninguna capa. Hoy el gate aplica automáticamente en toda location elegible y no se puede apagar por location. Es lo único del documento original que quedó fuera.
- **Landmine del hotel info:** los flags "Hotel information" de Commerce (`allowHotelRoom`/`allowExpectedDeparture`) se quitaron porque configuraban el mismo dato dos veces con semánticas distintas. Ese borrado (`7152ee8`) **existe solo en staging de Commerce**. La rama `feature/ticket-info-hotel-room` tiene un único commit — justo el que se quitó — así que si se mergea a `main` el grupo duplicado regresa. Las columnas siguen en la BD y ms-valet-service las sigue leyendo.
- Sin verificar end-to-end: un `PATCH` completo, las dos filas de historial (status 9 y 15) y que el `hotelRoom` sobreviva en RTDB. Ver [[reference-find-cof-eligible-ticket]] para el ticket de prueba.

Relacionado: [[feature-hotel-info-gate]], [[project-phone-gate-is-survey-driven]], [[project-boot-orchestration]].
