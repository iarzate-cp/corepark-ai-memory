---
name: CorePark Backend — Stack Tecnológico
description: Stack común y configuración de infraestructura de los 8 microservicios
type: project
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
**Stack común:**
- Java 17 + Maven (excepción: **ms-oauth-service usa Java 21**)
- Spring Boot 3.5.15 (ms-valet, ms-firebase, ms-oauth, ms-partner) o 3.5.14 (reports, backoffice, pms, notifications) — convergiendo a 3.5.15
- PostgreSQL compartida (dev: localhost:5432/coreparkdev, prod: RDS AWS us-east-2 `dev-corepark-db-cluster.cluster-cf84k1igemey.us-east-2.rds.amazonaws.com`)
- RabbitMQ con SSL (puerto 5671, exchange: `direct.amq.corepark`, user `rabbit-admin`)
- Docker (Amazon Corretto 17/21 Alpine, arm64)
- Springdoc OpenAPI 2.8.13 con Scalar para documentación

**Integraciones externas:**
- AWS: S3, SQS (FIFO), Lambda (URL shortener), RDS
- Firebase Realtime Database (canalizado vía ms-firebase-service)
- Claude AI API (modelo `claude-sonnet-4-6`) — solo en ms-pms-service para análisis de archivos de arrivals
- Slack (alertas y monitoreo)
- Google Maps API, Square (pagos), Twilio (SMS)
- PMS hoteleros: Opera (OHIP cloud + on-premise), StayNTouch, Guesty, Infor
- EV Charging: Way, Evha
- Agregadores parking: OCRA, SpotHero, ParkChirp
- OAuth 2.0 con JWT (nimbus-jose-jwt + bcprov-jdk18on) — emitido por ms-oauth-service

**Comunicación inter-servicios (Feign, en localhost dev):**
- ms-valet (9000) → payment (9001), firebase (9010), notifications (9006), pms (9005), backoffice (9004)
- ms-partner (9009) → payment (9001), firebase (9010), valet (9000)
- Otros servicios consumen el cliente OAuth `valet-client` / `partner-client` definido en ms-oauth-service

**ms-payment-service** (puerto 9001) está referenciado por Feign pero NO existe en `/Users/israel/Dev/Back-End/` — repo separado o pendiente de extracción.

**Why:** Información verificada el 2026-06-23 leyendo pom.xml y application.yml de cada servicio.

**How to apply:** Al sugerir cambios de infraestructura/dependencias, mantener servicios alineados en Spring Boot (preferir 3.5.15 como objetivo común). Recordar que ms-oauth-service requiere Java 21, no 17. Para nuevos servicios extraídos del monolito, replicar el patrón rabbit SSL + datasource compartido.
