---
name: CorePark Backend — Visión General
description: Descripción de los 8 microservicios en /Users/israel/Dev/Back-End y su rol funcional
type: project
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
Ocho microservicios Java/Spring Boot para la plataforma CorePark (parking management):

- **ms-valet-service** (puerto 9000): Núcleo operativo — check-in/check-out, transacciones, control de efectivo, cupones, EV charging, agregadores (OCRA, SpotHero, ParkChirp), StackParking, GPS. ~115 endpoints
- **microservice-reports** (puerto 9002): 34 tipos de reportes (CSV/Excel/PDF), scheduler, distribución por email. ~87 endpoints
- **ms-backoffice-service** (puerto 9004): Administración — catálogos, configuración, tarifas, pagos, encuestas, push, importación masiva, auditoría, CRM. ~251 endpoints (el más grande)
- **ms-pms-service** (puerto 9005): Integración PMS (Opera OHIP cloud/on-prem, StayNTouch, Guesty, Infor), archivos de llegadas con Claude AI. ~31 endpoints
- **ms-notifications-service** (puerto 9006): Hub centralizado — SMS Twilio, email dual, chat broadcast/direct con receipts, push Firebase. ~24 endpoints
- **ms-oauth-service** (puerto 9007): OAuth 2.0 Password Grant — emisión/validación JWT (nimbus-jose-jwt), recuperación de password, pincode login, credenciales mobile/web AWS, OTP comercio. ~14 endpoints. **NUEVO** (Junio 2026)
- **ms-partner-service** (puerto 9009): Portal de socios/hoteles — request-car, validaciones de ticket, compensaciones, página guest. ~27 endpoints. **NUEVO** (Junio 2026)
- **ms-firebase-service** (puerto 9010): Bridge a Firebase Realtime DB — push notifications, postings, tickets, EV charging events. ~15 endpoints. **NUEVO** (Junio 2026)

**Why:** Plataforma de gestión de valet parking para hoteles, con integración a PMS hoteleros y múltiples canales de pago/estacionamiento. Los servicios `ms-firebase`, `ms-oauth` y `ms-partner` fueron extraídos del monolito original como parte de la modernización (verificado Junio 2026).

**How to apply:** Cuando el usuario pida cambios, ten en cuenta que todos los servicios comparten la misma BD (PostgreSQL `coreparkdev`) y el mismo RabbitMQ (exchange `direct.amq.corepark`). Se comunican vía OpenFeign. Hay también un `ms-payment-service` referenciado por Feign en puerto 9001 que NO está en este directorio (presumiblemente repositorio separado o aún no extraído).
