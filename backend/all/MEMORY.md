# Memory Index

- [Rol del usuario](user_role.md) — Israel se está integrando al equipo backend de CorePark, objetivo: documentar el sistema
- [Iniciativa de documentación](project_documentation_initiative.md) — Documentar los 8 microservicios como parte del onboarding
- [CorePark Backend — Visión General](project_overview.md) — 8 microservicios: valet, reports, backoffice, pms, notifications, oauth (NUEVO), partner (NUEVO), firebase (NUEVO)
- [CorePark Backend — Stack Tecnológico](project_stack.md) — Java 17 (oauth=21), Spring Boot 3.5.14/3.5.15, PostgreSQL compartida, RabbitMQ SSL, AWS, Firebase, Claude AI
- ["Commerce" = frontend](project_commerce_frontend.md) — "Commerce" es el proyecto FRONTEND de CorePark, no un microservicio backend

## Feedback
- [Conteos verificados por grep](feedback_endpoint_counts.md) — Siempre usar grep sobre src/main para contar endpoints, no estimaciones
- [Java 21 en ms-oauth-service](feedback_java_version_oauth.md) — Solo ms-oauth usa Java 21; el resto sigue en Java 17
- [Acceso a Firebase vía ms-firebase-service](feedback_firebase_access_pattern.md) — Código NUEVO consume ms-firebase vía REST, no usa FirebaseDao directo

## Documentación detallada por microservicio
- [Docs ms-valet-service](docs_ms_valet_service.md) — Puerto 9000, ~115 endpoints, núcleo valet, EV, cupones, agregadores
- [Docs microservice-reports](docs_microservice_reports.md) — Puerto 9002, ~87 endpoints, 34 tipos de reportes, scheduler 5min
- [Docs ms-backoffice-service](docs_ms_backoffice_service.md) — Puerto 9004, ~251 endpoints, admin/config, catálogos, tarifas, Firebase, S3, Slack
- [Docs ms-pms-service](docs_ms_pms_service.md) — Puerto 9005, ~31 endpoints, Opera/StayNTouch/Guesty/Infor, Claude AI arrivals
- [Docs ms-notifications-service](docs_ms_notifications_service.md) — Puerto 9006, ~24 endpoints, SMS Twilio, email, chat con receipts
- [Docs ms-oauth-service](docs_ms_oauth_service.md) — Puerto 9007, Java 21, OAuth 2.0 JWT + credenciales + OTP [NUEVO]
- [Docs ms-partner-service](docs_ms_partner_service.md) — Puerto 9009, portal socios/hoteles + guest page, request-car [NUEVO]
- [Docs ms-firebase-service](docs_ms_firebase_service.md) — Puerto 9010, bridge Firebase Realtime DB, postings, tickets, EV [NUEVO]
