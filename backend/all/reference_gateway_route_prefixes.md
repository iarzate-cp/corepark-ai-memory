---
name: ms-gateway-service — rutas por prefijo y StripPrefix
description: Lista actual de rutas del gateway, qué servicio atienden y si strippean prefijo; qué hacer cuando un controller vive bajo un prefijo no ruteado
type: reference
---
## Rutas actuales

`ms-gateway-service/src/main/resources/application-dev.yml` y `application-prod.yml` (sincronizadas en su lista de rutas):

| Prefijo         | Target                     | StripPrefix |
|-----------------|----------------------------|-------------|
| `/backoffice/**`  | ms-backoffice-service     | 1           |
| `/credentials/**` | ms-oauth-service          | 0           |
| `/reports/**`     | ms-reports-service        | 1           |
| `/security/**`    | ms-oauth-service          | 1           |
| `/payment/**`     | ms-payment-service        | 1           |
| `/partner/**`     | ms-valet-service          | 0           |
| `/valet/**`       | ms-valet-service          | 1           |
| `/crm/**`         | ms-backoffice-service     | 0           |
| `/notifications/**` | ms-notifications-service | 1         |
| `/pms/**`         | ms-pms-service            | 1           |

## Prefijos NO ruteados

- `/validations/**` — aunque `ms-valet-service` tiene `@RequestMapping("validations")` en `ValidationsController`, ese prefijo no está en el gateway.
- `/catalogs/**` — `ms-valet-service` expone `GET /catalogs/country-codes`, pero el prefijo no está en el gateway.

## Cómo llegarles

Prefijar con `/valet/` en el cliente. El gateway hace `StripPrefix=1`, así el servicio recibe la ruta original:

- Cliente: `GET /valet/catalogs/country-codes` → servicio: `GET /catalogs/country-codes` ✓
- Cliente: `PUT /valet/validations/ticket-info/{ticket}` → servicio: `PUT /validations/ticket-info/{ticket}` ✓

Alternativa "limpia" que no elegimos: agregar rutas `/validations/**` y `/catalogs/**` al `application-*.yml` del gateway. Requiere PR y coordinación; el prefijo `/valet/` funciona sin tocar shared infra.

## Trampas conocidas

- **`/partner/**` no strippea el prefijo** (deliberado: los endpoints partner-hotel esperan el prefijo).
- **`/reports/**` va a `ms-reports-service`**, no a `ms-valet-service`. Endpoints que uno esperaría del valet-service (ticket detail, overnight summary) los sirve reports-service, con su propio DAO. Un cambio en `ms-valet-service` no se refleja ahí automáticamente.
- **CORS**: la lista `allowed-origins` incluye `https://localhost:4000/4100/4300/4400/4600` en dev. HTTP variants **no** están; si el frontend corre en HTTP local necesita agregar el origin manualmente al yml.
