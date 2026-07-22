---
name: Docs ms-firebase-service
description: Bridge a Firebase Realtime DB — push notifications, postings, tickets, EV charging. Puerto 9010
type: project
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
**Puerto:** 9010 · **Stack:** Java 17 + Spring Boot 3.5.15 · **App name:** `firebase-service`

**Responsabilidad:** Bridge centralizado a Firebase Admin SDK. Otros servicios envían mensajes vía RabbitMQ o REST y este servicio los publica a Firebase Realtime Database. También recibe webhooks de EV charging.

**Estructura por dominio (`com.corepark.firebase.*`):**
- `controller/FirebaseController` — endpoints generales: `/health`, `sendMQMessage`, `processPosting`, `firebase/process-posting`, `firebase/process-ticket`, `firebase/send-notification`
- `evcharging/` — eventos de carga eléctrica: POST `/events`, PUT `/tickets/{ticket}`, POST `/cleanup`
- `notification/` — request push: POST/PUT `request`
- `posting/` — postings hoteleros: GET `/posting`, DELETE `/posting/`, PUT `/posting/process-at-front-desk`
- `ticket/` — GET `/ticket/carlist`

**Dependencias clave:** firebase-admin, grpc-netty-shaded, postgresql, spring-amqp.

**Integraciones:** Firebase Realtime DB (autenticación vía admin SDK), RabbitMQ listener (cola `firebase-realtime` en exchange `direct.amq.corepark`).

**Why:** Servicio extraído del monolito (~Junio 2026). Centraliza el contacto con Firebase para que el resto de microservicios no carguen el SDK ni credenciales.

**How to apply:** Si algún servicio necesita publicar a Firebase, debe llamar este servicio vía Feign (`ms-firebase-service` en `http://localhost:9010`) o publicar a la cola RabbitMQ apropiada. NO duplicar el SDK de Firebase en otros servicios.
