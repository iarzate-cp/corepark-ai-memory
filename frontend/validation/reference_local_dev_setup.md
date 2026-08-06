---
name: Local dev setup — frontend-validation contra backend local
description: Cómo correr frontend-validation apuntando a ms-valet-service local (puerto 8080 vía gateway 4400)
type: reference
---
## Archivos clave

- `src/environments/environment.local.ts` — clon del `dev` con `endpoint: 'https://localhost:8080'`. Firebase config apunta a `corepark-services-dev` (mismo que dev, sin duplicar backend Firebase local).
- `package.json` → script `start:local`: `ng serve --port 4400 --ssl --configuration=local --poll 2000 --no-hmr`.
- `angular.json` → configuration `local` bajo `architect.build.configurations` con `fileReplacements` (env.ts → env.local.ts). También registrada bajo `architect.serve.configurations`.

## Por qué puerto 4400 con --ssl

- Elegido porque `https://localhost:4400` ya está en la whitelist de CORS del gateway (`ms-gateway-service/application-dev.yml:104`), no hace falta pedirle al backend agregar orígenes.
- `--ssl` es necesario porque **mixed content** bloquea que un frontend HTTPS llame a un backend HTTP. Y `--poll 2000 --no-hmr` va sincronizado con el `start` normal para consistencia con setups de Docker/network mount.
- El backend local (`ms-valet-service`) escucha en `http://localhost:8080` (**HTTP**, no HTTPS). El gateway no tiene SSL server-side (el `ssl.enabled: true` en el yml es para RabbitMQ, no para el server). Combinar `--ssl` en el front + `http://` en el back **funciona en la práctica en localhost** aunque el modelo mental sea "mixed content"; algunos navegadores lo permiten en localhost, otros piden `chrome://flags/#allow-insecure-localhost`. Si se rompe, la alternativa limpia sería habilitar SSL en el gateway (keystore + server.ssl).

## Prefijo /valet/ obligatorio en los HttpPath

El gateway solo tiene rutas para:
- `/backoffice/**`, `/credentials/**`, `/reports/**`, `/security/**`, `/payment/**`, `/partner/**`, `/valet/**`, `/crm/**`, `/notifications/**`, `/pms/**`.

**No rutea `/validations/**` ni `/catalogs/**` directamente.** Como el valet-service tiene los controllers `@RequestMapping("validations")` y `catalogs/*` sin prefix a nivel de clase, la única forma de llegarles desde el gateway es prefijando con `/valet/` en el frontend:

- `HttpPath.ValidationsTicketInfo = '/valet/validations/ticket-info'`
- `HttpPath.CountryCodes           = '/valet/catalogs/country-codes'`

El gateway matchea `/valet/**` con `StripPrefix=1`, así que el servicio recibe `/validations/...` y `/catalogs/...` como si no hubiera prefijo.

## Ticket para el DB local

- Postgres local en `postgresql://dewsdbcp@localhost:5432/coreparkdev`.
- El tunnel a la DB compartida (Slack) puede confundir: si tu servicio va contra la DB de otro entorno, verás datos inesperados. Verificar primero el tunnel/config antes de asumir bugs en el código.

## Notas del bootstrap

- `provideNotifications({ position: 'top-right' })` está registrado globalmente en `app.config.ts` — cualquier componente puede inyectar `NotificationService` de corepark-ui y llamar `.show({ type, title, description })`.
- El interceptor `operatorBodyInterceptor` inyecta `operatorCompanyId`+`parkingLocationId` a los bodies de POST/PUT. Bypass agregando el path a `NOT_ALLOWED_PATH_LIST` cuando el endpoint recibe eso por headers.
