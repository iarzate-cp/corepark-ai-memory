---
name: Documentación técnica ms-notifications-service
description: Documentación detallada del microservicio centralizado de notificaciones SMS, email y chat (puerto 9006)
type: project
originSessionId: fc4c993d-0479-4108-8ad1-e63c64662c52
---
# ms-notifications-service

## 1. Propósito y contexto
Hub centralizado de comunicación multicanal de CorePark. Gestiona 3 canales:
- **SMS:** vía Twilio (envío individual, masivo, recibos, webhook entrante)
- **Email:** vía Gmail SMTP con 2 cuentas (noreply y reporting)
- **Chat:** conversaciones en tiempo real entre Validation Portal (VP) y dispositivos Android (valets), con receipts de entrega/lectura

También recibe eventos asíncronos vía RabbitMQ (queue `sms-service`) y envía push notifications a dispositivos vía ms-firebase-service.

Contexto multi-tenant: cada request incluye `Operator-Id` + `Location-Id` en headers.

## 2. Tech stack
- **Java 17 · Spring Boot 3.5.14 · Spring Cloud 2025.0.2 · Maven**
- **Artifact:** `com.corepark.notificactions:NotificationsService:1.0`
- **Puerto dev:** 9006 · **Puerto prod:** 80
- **Twilio SDK 10.9.2** · **Spring Boot Mail + Thymeleaf** (templates HTML)
- **Spring Boot AMQP** (RabbitMQ consumer) · **Spring Cloud OpenFeign**
- **PostgreSQL 42.7.11 · Springdoc OpenAPI 2.8.13 · Lombok 1.18.38**

## 3. Arquitectura y patrones
Capas: **Controller → Service (interfaz + impl) → DAO (NamedParameterJdbcTemplate)**

Paquetes principales:
```
com.corepark.notifications
├── sms/
│   ├── controller/    (SmsController — 9 endpoints)
│   ├── service/       (SmsService + TwilioService)
│   ├── dao/           (ParkingServiceDao, TwilioAuditDao)
│   ├── senders/       (TwilioService — SDK integration)
│   ├── listener/      (AsyncListener — RabbitMQ consumer "sms-service")
│   └── beans/         (DTOs + enums)
├── email/
│   ├── controller/    (EmailController — 2 endpoints)
│   ├── service/       (EmailService)
│   └── config/        (ThymeleafConfig)
├── chat/
│   ├── controller/    (ChatController — 8 endpoints)
│   ├── service/       (ChatService)
│   ├── dao/           (ChatDao — conversations, messages, receipts)
│   ├── service/event/ (ChatNotificationEvent + listener)
│   └── beans/         (ConversationType: BROADCAST/DIRECT)
├── feign/
│   ├── IFirebaseFeignService  (push notifications)
│   └── IValetFeignService     (datos de tickets)
├── guestprofile/      (GuestProfileService, GuestProfileDao)
├── utils/             (MessageCode, IntentDetector, StringMatchingUtils)
├── config/            (RabbitMQConfig, MailConfig — 2 senders, OpenApiConfig)
└── commons/           (BaseController, RestExceptionHandlerController, ServiceException)
```

Patrones: Service/DAO, RabbitMQ @RabbitListener (async), TransactionalEventListener (chat notifications after commit), FeignClient para Firebase y Valet.

## 4. Endpoints
**Total: 20 endpoints** (verificado con grep sobre src/main, excluyendo Feign interfaces)

### SmsController (`/sms`)
| Método | Ruta | Descripción | Headers | Body/Params |
|--------|------|-------------|---------|-------------|
| POST | `/sms/twilio/webhook/{operatorCompanyId}/{parkingLocationId}` | Webhook Twilio SMS entrante | - | form-encoded: Body, From, To, MessageSid |
| POST | `/sms/send` | Enviar SMS genérico | - | `GenericSmsBean`: to, message, from (opt), operatorCompanyId, parkingLocationId, smsUsage, ticket |
| POST | `/sms/send-single-sms` | Enviar SMS individual | `operator-id`, `location-id` | `GenericSmsBean` |
| POST | `/sms/send-receipt` | Enviar recibo de pago por SMS | `operator-id`, `location-id` | `ReceiptSMS`: paymentId, userPhoneNumber (E.164), receiptType |
| POST | `/sms/send-bulk-sms` | Enviar SMS masivo a lista de tickets | `operator-id`, `location-id` | `BulkSmsRequestBean`: body, ticketNumbers, smsUsage |
| POST | `/sms/send-message-template` | Enviar SMS desde template | - | `NotificationServiceSMSTemplate`: operatorId, locationId, ticket, templateName, variables (Map), requesterPhoneNumber |
| POST | `/sms/send-expected-delivery` | Notificar tiempo estimado de entrega | - | `ExpectedDeliveryRequestBean`: operatorCompanyId, parkingLocationId, ticket, minutes |
| GET | `/sms/get-sms-chat-history` | Historial SMS de un ticket | `operator-id`, `location-id` | query: `ticket` |
| POST | `/sms/mark-message-read` | Marcar mensajes como leídos | `Operator-Id`, `Location-Id` | `SmsNotificationStatus` |

**Webhook Twilio — intención detectada (`IntentDetector`):**
- Keyword de solicitud de auto (ej: "CAR") → crea request de vehículo vía Valet Feign
- Chat genérico → almacena en historial SMS
- Selección de vehículo → guest profile selection

### EmailController (`/email`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/email/reports/overnight` | Enviar reporte como adjunto email (multipart: file + query params: operatorName, locationName, reportName, recipients[], fromPeriod, toPeriod, hourPeriod) |
| GET | `/email/test` | Endpoint de prueba hardcodeado (NO usar en prod) |

**Email sender alternation:** alterna entre `noreply@corepark.com` y `reporting@corepark.com` para load balancing.

### ChatController (`/chat`)
| Método | Ruta | Descripción | Headers |
|--------|------|-------------|---------|
| GET | `/chat/conversations` | Listar conversaciones del participante | `Operator-Id`, `Location-Id` + query: `androidId` o `requestPointUuid` |
| GET | `/chat/conversations/{conversationUuid}/messages` | Mensajes de conversación con receipts | `Operator-Id`, `Location-Id` |
| POST | `/chat/conversations` | Crear conversación + mensaje inicial (BROADCAST o DIRECT, idempotente) | `Operator-Id`, `Location-Id` |
| POST | `/chat/messages` | Enviar mensaje en conversación existente | `Operator-Id`, `Location-Id` |
| GET | `/chat/messages/{messageUuid}/receipts` | Receipts de un mensaje (PENDING/SENT/DELIVERED/READ) | `Operator-Id`, `Location-Id` |
| GET | `/chat/devices` | Dispositivos Android en la ubicación | `Operator-Id`, `Location-Id` |
| GET | `/chat/notifications` | Notificaciones para Android app (últimas 24h) | `Operator-Id`, `Location-Id` |
| PUT | `/chat/messages/{messageUuid}/acknowledge` | Marcar mensaje como leído | `Operator-Id`, `Location-Id` |

### CommonController
| Método | Ruta |
|--------|------|
| GET | `/health` |

## 5. Lógica de negocio principal

### Envío de SMS (flujo simple)
`/sms/send-single-sms` → `SmsService.sendSMS()` → `TwilioService.sendSingleSMS()` → Twilio SDK → registra auditoría en BD → retorna `ResponseBean`

### Envío de SMS masivo
`/sms/send-bulk-sms` → para cada ticket: obtiene phoneNumber de `company.parking_service`, llama `TwilioService.sendBulkSMS()` → retorna `BulkSMSResponse` con status por ticket

### Webhook Twilio entrante
1. Recibe form-encoded: `Body`, `From`, `To`, `MessageSid`
2. `IntentDetector` analiza el texto → detecta intención (car request, chat, vehicle selection)
3. Según intención: crea request de auto vía `IValetFeignService.requestCar()` / almacena en historial SMS
4. Responde 200 OK (Twilio requiere respuesta < 15s)

### Envío de email con adjunto (reports)
`/email/reports/overnight` (multipart) → `EmailService.sendOvernightReport()` → carga template Thymeleaf `overnight-report` → `JavaMailSender.send()` con `MimeMessage` + adjunto → alterna entre sender noreply y reporting

### Ciclo de vida de un mensaje de chat
1. **Crear conversación** (`POST /chat/conversations`):
   - BROADCAST: 1 por ubicación (reutiliza si existe), sincroniza nuevos dispositivos
   - DIRECT: 1 por par de participantes (reutiliza si existe)
2. **Enviar mensaje** (`POST /chat/messages`): inserta en `chat.messages` + crea receipts en `chat.message_receipts` (todos PENDING inicialmente)
3. **`@TransactionalEventListener`** (después del commit): publica `ChatNotificationEvent` → `IFirebaseFeignService.sendNotification()` → push a dispositivos
4. **Receipts evolucionan:** PENDING → SENT (entregado a Firebase) → DELIVERED (app recibió) → READ (usuario leyó)
5. **Acknowledge** (`PUT /chat/messages/{uuid}/acknowledge`): marca receipt como READ

### Async RabbitMQ (queue `sms-service`)
`AsyncListener` consume `GenericSmsBean` o `GenericEmailBean` → invoca SmsService / EmailService correspondiente

## 6. Base de datos
**Dev:** `jdbc:postgresql://localhost:5432/coreparkdev` · user: `dewsdbcp` · pass: `xxv87htXui8iHm`
**Prod:** RDS us-east-2 `corepark` · user: `prwsdbcp`
**Pool Hikari:** min 2, max 5, idle 10min

Tablas principales:
| Tabla | Descripción |
|-------|-------------|
| `company.parking_service` | Tickets (phoneCode, phoneNumber para SMS) |
| `custom.cat_country` | Códigos de país/teléfono |
| `chat.conversations` | Conversaciones (uuid PK, type: BROADCAST/DIRECT, operator_id, location_id) |
| `chat.conversation_participants` | Participantes de conversación |
| `chat.participants` | Registro de participantes (devices + request points) |
| `chat.messages` | Mensajes (uuid PK, conversation_uuid, content, sender_participant_uuid, created_at) |
| `chat.message_receipts` | Estado de entrega (message_uuid, participant_uuid, status: PENDING/SENT/DELIVERED/READ) |

## 7. Integraciones

### Feign Clients
| Cliente | Servicio | URL dev | Propósito |
|---------|----------|---------|-----------|
| IFirebaseFeignService | ms-firebase-service | localhost:9010 | Push notifications: `/notification/request`, `/firebase/send-notification` |
| IValetFeignService | ms-valet-service | localhost:9000 | Crear solicitud de auto al detectar SMS: `/tickets/request` |

Timeout: connectTimeout 60s, readTimeout 62s

### RabbitMQ
- **Host dev:** AWS MQ dev (b-c42715a3....mq.us-east-2.amazonaws.com:5671, SSL)
- **Host prod:** AWS MQ prod
- **Exchange:** `direct.amq.corepark`
- **Queue `sms-service`** → `AsyncListener` consume `GenericSmsBean`/`GenericEmailBean`

### Twilio SMS
- **Account SID:** `<REDACTED — véase Twilio console / 1Password>`
- **Número:** +15045178870 (California)
- **Webhook URL pattern:** `/sms/twilio/webhook/{operatorCompanyId}/{parkingLocationId}`

### Gmail SMTP
| Cuenta | Propósito | SMTP |
|--------|-----------|------|
| `noreply@corepark.com` | SMS receipts, reportes (bean: `emailSenderNoReply`) | smtp.gmail.com:465 SSL |
| `reporting@corepark.com` | Reportes de reporting (bean: `emailSenderReporting`) | smtp.gmail.com:465 SSL |

### Firebase
- Delegado a ms-firebase-service vía Feign (`IFirebaseFeignService`)
- Usado para push notifications después de crear mensajes de chat

## 8. Configuración por entorno
| Propiedad | Dev (local) | Dev (AWS) | Prod |
|-----------|-------------|-----------|------|
| port | 9006 | 80 | 80 |
| DB | localhost:5432/coreparkdev | AWS RDS dev | AWS RDS prod |
| RabbitMQ | localhost:5671 | AWS MQ dev | AWS MQ prod |
| Firebase feign | localhost:9010 | ms-firebase-service.${NAMESPACE} | ms-firebase-service.internal-corepark.com |
| Valet feign | localhost:9000 | ms-valet-service.${NAMESPACE} | ms-valet-service.internal-corepark.com |

Variables de entorno (K8s): `NAMESPACE`, `CLUSTER_NAME`

**Nota CI:** `application-ci.yml` excluye JDBC, RabbitMQ, Mail, Scheduling del autoconfigure (para tests sin infraestructura)

## 9. Cómo correr localmente

**Prerequisitos:**
- Java 17, Maven, PostgreSQL local (coreparkdev), RabbitMQ local
- Credenciales Gmail app passwords (en application.yml: `noreply` y `reporting`)
- Twilio credentials (Account SID + Auth Token)

**Build y run:**
```bash
cd /Users/israel/Dev/Back-End/ms-notifications-service
mvn clean package -DskipTests
java -jar target/NotificationsService-1.0.jar --spring.profiles.active=dev
# O con Maven:
./mvnw spring-boot:run
```

**Verificar:**
```bash
curl http://localhost:9006/health
# {"status":"UP"}
```

**Otros MS requeridos para funcionalidad completa:**
- ms-firebase-service (:9010) para push notifications en chat
- ms-valet-service (:9000) para procesar solicitudes de auto vía webhook SMS

**Nota:** Sin RabbitMQ corriendo, el `AsyncListener` no arrancará. Sin Twilio credentials, el envío de SMS fallará.
