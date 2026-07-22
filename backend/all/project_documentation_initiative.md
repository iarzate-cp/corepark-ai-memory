---
name: Iniciativa de documentación del backend CorePark
description: Israel está liderando la documentación de los 8 microservicios como parte de su onboarding al equipo
type: project
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
Israel está documentando los microservicios de CorePark como parte de su integración al equipo de backend.

**Proyectos a documentar (8 microservicios, actualizado 2026-06-23):**
- ms-valet-service (9000) — núcleo operativo
- microservice-reports (9002) — reportes
- ms-backoffice-service (9004) — admin/config
- ms-pms-service (9005) — integración PMS hoteleros
- ms-notifications-service (9006) — hub notificaciones
- ms-oauth-service (9007) — OAuth/JWT/credenciales [NUEVO Jun-2026]
- ms-partner-service (9009) — portal socios/guest [NUEVO Jun-2026]
- ms-firebase-service (9010) — bridge Firebase [NUEVO Jun-2026]

**Why:** Onboarding al equipo — necesita entender y dejar registro del sistema para él y futuros integrantes. Existen además 3 servicios nuevos (firebase, oauth, partner) extraídos del monolito recientemente que aún no tienen documentación interna.

**How to apply:** Cuando ayude con documentación, orientarla como material de onboarding. Priorizar: propósito del servicio, cómo corre localmente, endpoints clave, integraciones entre servicios, y decisiones de arquitectura no obvias. Para los servicios nuevos (firebase/oauth/partner), enfatizar que son extracciones del monolito original — la motivación arquitectónica es relevante.
