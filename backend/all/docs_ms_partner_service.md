---
name: Docs ms-partner-service
description: Portal de socios/hoteles — request-car, validación tickets, compensaciones, página guest. Puerto 9009
type: project
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
**Puerto:** 9009 · **Stack:** Java 17 + Spring Boot 3.5.15 · **App name:** `ms-partner-service`

**Responsabilidad:** Lógica del lado de socios (hoteles partner) y página pública de guest. Maneja request-car desde el hotel, validación de tickets con descuentos hoteleros, compensaciones (comps), y la página web que ve el huésped para pedir su coche y pagar.

**Estructura por dominio (`com.corepark.partner.*`):**

- **`hotel/HotelController`** (~3 endpoints) — PUT `hotel/ticket-info`, POST `hotel/announced-departure`, POST `hotel/unannounced-departure`
- **`partner/PartnerController`** (~13 endpoints) — `request-car` (+ deprecated `send-request-car`), `cancel-request`, `get-validation-info`, `ticket-validation`, `get-validation-details`, `validate-ticket`, `compensation`, `get-validations`, `get-requests`, `leaving-times/{operatorCompanyId}/{parkingLocationId}`, `/health`
- **`guest/GuestPageController`** (~5+ endpoints) — `/ticket-info/{parkingLocationUUID}/{ticket}`, `/request-car`, `/cancel-request`, `/payment/get-detail`, `/payment/pay-ticket` (página pública para el huésped)
- **`payment/`** — Feign a ms-payment-service (Square)
- **`coupon/`** — Feign a ms-valet-service para cupones
- **`firebase/`** — Feign a ms-firebase-service
- **`rabbit/`** — listeners RabbitMQ

**Feign clients:**
- `ms-payments-service` → `http://localhost:9001`
- `ms-valet-service` (cupones) → `http://localhost:9000`
- `ms-firebase-service` → `http://localhost:9010`

**Dependencias clave:** spring-cloud-starter-openfeign, spring-cloud-starter-loadbalancer, postgresql, commons-fileupload, bcprov-jdk18on.

**Why:** Servicio NUEVO (Junio 2026). Extraído del monolito para separar la superficie pública (guest) y partner del núcleo operativo valet.

**How to apply:** Cualquier funcionalidad expuesta al hotel partner o al huésped final cae aquí. Si una request-car llega desde el front-desk del hotel, este servicio orquesta: valida ticket → llama valet vía Feign → publica a Firebase. Hay endpoints DEPRECATED (ej. `send-request-car`) — preferir las versiones nuevas (`request-car`).
