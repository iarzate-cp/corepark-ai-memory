---
name: Acceso a Firebase desde otros microservicios va vía ms-firebase-service
description: Para código NUEVO, no usar FirebaseDao directo de ms-backoffice; siempre consumir ms-firebase-service vía REST
type: feedback
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
Aunque **ms-backoffice-service** tiene acceso directo a Firebase RTDB vía `com.corepark.backoffice.common.dao.FirebaseDao` (ya usado por `RateDaoImpl`, `PushMessageDaoImpl`), **para código nuevo que necesite leer/escribir Firebase desde backoffice (u otros servicios) se consume `ms-firebase-service` vía REST**, no se usa `FirebaseDao` directo.

**Why:** Decisión arquitectónica acordada con Israel el 2026-06-24. `ms-firebase-service` (puerto 9010) es la abstracción que centraliza paths, índices y estructura de RTDB. Si cada microservicio empieza a tocar directamente paths de Firebase, esa abstracción pierde sentido y se duplica el conocimiento del schema RTDB. El "salto de red extra" es despreciable para endpoints administrativos. `FirebaseDao` en backoffice existe por razones legacy.

**How to apply:**
- Para features nuevos que toquen Firebase desde backoffice (o cualquier otro servicio), agregar/consumir endpoints en `ms-firebase-service` (ej. `GET /ticket/carlist`, `POST /v1/ev-charging/cleanup`).
- No agregar nuevos DAOs que importen `FirebaseDatabase`/`DatabaseReference` directamente en otros microservicios.
- Si necesitas una operación que aún no expone ms-firebase, primero abre endpoint en ms-firebase, luego consúmelo.
- Excepción: si Israel pide explícitamente la ruta directa por urgencia/rendimiento, confirmar con él y dejar nota en el código.
