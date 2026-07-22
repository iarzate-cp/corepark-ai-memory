---
name: Documentación técnica ms-pms-service
description: Documentación detallada del microservicio de integración con sistemas PMS hoteleros (puerto 9005)
type: project
originSessionId: fc4c993d-0479-4108-8ad1-e63c64662c52
---
# ms-pms-service

## 1. Propósito y contexto
Gateway de integración entre CorePark y sistemas PMS hoteleros (Property Management Systems). Centraliza la comunicación con hoteles para: post-to-room (cargar tickets a habitaciones), obtención de huéspedes activos, procesamiento de archivos de llegadas, polling automático de Opera Cloud, y recepción de webhooks de PMS.

Soporta 5 sistemas PMS: **Opera Cloud (OHIP)**, **Opera On-Premise**, **StayNTouch**, **Guesty**, **Infor**.

Usa **Claude AI** para parsear archivos de llegadas en cualquier formato (PDF, CSV, TXT, XLS).

## 2. Tech stack
- **Java 17 · Spring Boot 3.5.14 · Spring Cloud 2025.0.2 · Maven**
- **Artifact:** `com.corepark.pms:PMSService:1.0`
- **Puerto dev:** 9005 · **Puerto prod:** 80
- **Spring Boot AMQP, JDBC, Mail, Web, Validation**
- **Spring Cloud OpenFeign + Load Balancer**
- **AWS Spring Cloud SQS 3.2.1**
- **Anthropic Claude Java SDK 2.32.0** (parseo AI de archivos)
- **Apache PDFBox 3.0.6, POI 5.5.1, OpenCSV 5.12.0**
- **Slack API Client 1.44.2** · **PostgreSQL 42.7.11**

## 3. Arquitectura y patrones
Capas: **Controller → Service (interfaz + impl) → DAO (JDBC directo)**

Paquetes principales:
```
com.corepark.pms
├── controller/         (PmsController — endpoints generales)
├── service/            (IPmsService, SchedulerService, PaymentService)
├── dao/                (consultas JDBC)
├── config/             (ThreadPool, SQS, Scheduling, Jackson, CiMockConfig)
├── types/              (enums: PmsType, OperaEventType, MessageCode, RabbitMQ keys)
├── arrivals/           (pipeline de archivos de llegadas)
│   ├── controller/     (ArrivalsController)
│   ├── service/        (ArrivalsFileValidatorService, ArrivalsFileApplyService)
│   └── dao/            (PmsArrivalsFileConfigDao, PmsArrivalsFileProcessingDao)
├── claude/             (ClaudeApiClient, ClaudeFileParserService, ClaudeApiProperties)
├── opera/
│   ├── cloud/          (OhipController, OhipService, OhipApiRestService)
│   └── onpremise/      (OperaOnPremiseController, SQSMessageListenerService)
├── stayntouch/         (StayNTouchController, StayNTouchService, StayNTouchDao)
├── guesty/             (GuestyController, GuestyService — HMAC Svix validation)
├── infor/              (InforController, InforService — SOAP XML)
├── slack/              (SlackNotificationService)
├── feign/              (IFirebaseFeignService)
├── rabbit/             (RabbitService — publica MQPosting, MQTicket)
└── sqs/                (SQSMessageListenerService)
```

Patrones: Strategy (cada PMS tiene su impl), Message-Driven (RabbitMQ primario + HTTP fallback), @Scheduled para polling y autoposting, @SqsListener para Opera On-Premise.

**PmsType enum:**
- 1 = OPERA_CLOUD
- 2 = OPERA_ON_PREMISE
- 3 = GUESTY
- 4 = INFOR
- 5 = STAY_N_TOUCH
- 6 = LIGHT (sin PMS real)

## 4. Endpoints
**Total: 29 endpoints** (verificado con grep sobre src/main, excluyendo Feign interfaces; incluye OperaLegacyController deprecated)

### PmsController (`/`)
| Método | Ruta | Descripción | Headers |
|--------|------|-------------|---------|
| GET | `/health` | Health check | - |
| POST | `/validate-posting` | Validar elegibilidad de post-to-room (body: operatorCompanyId, parkingLocationId, partnerId, room, lastName) | - |
| GET | `/partners` | Partners PMS de una ubicación | `Operator-Id`, `Location-Id` |
| GET | `/guests/{pmsTypeId}` | Buscar huéspedes por filtro global | `Operator-Id`, `Location-Id`, `Partner-Id` |
| GET | `/guests/eligible` | Huéspedes elegibles para post-to-room (<30 días llegada, salida >-48h) | `Operator-Id`, `Location-Id` |

### ArrivalsController (`/arrivals`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/arrivals/validate` | Validar archivo de llegadas con Claude (multipart: file, fileFormat) sin persistir |
| POST | `/arrivals/apply` | Parsear y persistir huéspedes (multipart: file, configId) |

### OhipController — Opera Cloud (`/opera/cloud`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/opera/cloud/post-to-room` | Postear cargo a habitación |
| POST | `/opera/cloud/guest-posting` | Asociar huésped a ticket |
| POST | `/opera/cloud/test-ocim-token` | Validar credenciales OCIM OAuth |
| POST | `/opera/cloud/{gatewayId}/test-external-code` | Probar código de sistema externo |
| POST | `/opera/cloud/test-ssd-token` | Validar credenciales SSD OAuth |

### OperaOnPremiseController (`/opera/on-premise`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/opera/on-premise/post-to-room` | Postear cargo a habitación |
| POST | `/opera/on-premise/void-posting` | Anular posting |
| POST | `/opera/on-premise/guest-posting` | Asociar huésped |

### StayNTouchController (`/stayntouch`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/stayntouch/webhook` | Webhook StayNTouch (almacena asíncronamente) |
| POST | `/stayntouch/post-to-room` | Postear cargo a habitación |
| GET | `/stayntouch/posting-accounts` | Cuentas de posteo disponibles (Headers: Operator-Id, Location-Id) |
| POST | `/stayntouch/post-to-account` | Postear a cuenta (no a habitación) |

### GuestyController (`/guesty`)
| Método | Ruta | Descripción | Headers |
|--------|------|-------------|---------|
| POST | `/guesty/webhook/{uuid}` | Webhook Guesty con validación HMAC Svix | `svix-id`, `svix-timestamp`, `svix-signature` |
| POST | `/guesty/post-to-room` | Postear cargo a habitación | - |

### InforController (`/infor`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/infor/post-to-room` | Postear cargo a folio |
| POST | `/infor/webhook` | Webhook Infor SOAP (OTA_HotelResNotif XML → JAXB → OTA_HotelResNotifRS response) |

### OperaLegacyController (`/opera/legacy`) — DEPRECATED
Mismo set que OperaOnPremise. Marcado `@Deprecated`.

## 5. Lógica de negocio principal

### Post-to-Room (cargos a habitaciones)
1. Obtiene config PMS y credenciales del gateway
2. Busca reservación por `lastName + room`
3. Para cada ticket: crea UUID de transacción, inserta en `payment_transaction`, llama REST a PMS externo, almacena resultado en `pms_data_posting`
4. Publica `MQPosting` vía RabbitMQ (fallback: Feign HTTP a firebase)

### Procesamiento de archivos de llegadas (arrivals pipeline)
1. **Email Reading** (`EmailHelperService`): IMAP a Gmail `operaarrivals@corepark.com`, extrae adjuntos con subject pattern matching
2. **Parsing con Claude AI** (`ClaudeFileParserService`):
   - PDF → `Base64PdfSource`; CSV/TXT → texto plano
   - System prompt: extrae `firstName|lastName|accompanyingGuest|arrivalDate|departureDate|room|reservationId|status` en formato pipe-delimited
   - Modelo: `claude-sonnet-4-6`, 32k tokens, timeout 180s, 2 reintentos con backoff
   - Si retorna `ERROR|reason` → `ClaudeErrorKind.ERROR` → alerta Slack
3. **Persistencia** (`ArrivalsFileApplyService`): inserta en `pms_data` con status PENDING; registra en `pms_arrivals_file_processing`

### OHIP Polling (Opera Cloud)
- `OhipUpdaterService` arranca con `ApplicationReadyEvent`
- Agenda tasks en `ohipPollingTaskScheduler` (2 threads) con `fixedDelay` por hotel
- Cada tick: obtiene nuevo token OAuth, llama `/v1/checkins` (Business Events), parsea `ReservationInfo`, inserta/actualiza `pms_data` y `opera_cloud_data`

### Autoposting Scheduler
`SchedulerService.executePendingPayments()` cada **5 min** (`@Scheduled`):
- Consulta `pms_data WHERE status='PENDING'` del último día
- Para cada pending: switch por PmsType → llama servicio específico (Ohip, OperaOnPremise, StayNTouch, etc.)
- Thread pool: `autopostingTaskExecutor` (1-2 threads, queue 25, CallerRunsPolicy)

### Webhooks
- **Guesty:** valida HMAC-SHA256 Svix (`svix-id.svix-timestamp.payload`) antes de procesar
- **StayNTouch:** almacena en log, retorna 200 inmediatamente, procesa asíncronamente
- **Infor:** unmarshal SOAP XML con JAXB, almacena, retorna SOAP response estándar
- **SQS (Opera On-Premise):** `@SqsListener` en cola FIFO `hotel-responses-{location-uuid}.fifo`, ack ON_SUCCESS

## 6. Base de datos
**Dev:** `jdbc:postgresql://localhost:5432/coreparkdev` · user: `dewsdbcp` · pass: `xxv87htXui8iHm`
**Prod:** RDS us-east-2 `corepark` · user: `prwsdbcp`
**Pool Hikari:** min 2, max 5, idle 10min

Tablas principales:
| Tabla | Descripción |
|-------|-------------|
| `pms_data` | Huéspedes importados de PMS (id, operator_id, location_id, partner_id, guest_data_json, pms_type_id, status: PENDING/APPLIED/FAILED) |
| `payment_transaction` | Transacciones de pago (uuid PK, amount, currency, timestamp) |
| `parking_service_payment` | Link payment_transaction ↔ ticket (posting_status, response_code) |
| `pms_arrivals_file_cfg` | Config de archivos de llegadas (file_format, subject_pattern_regex, active) |
| `pms_arrivals_file_processing` | Auditoría de procesamiento (status: SUCCESS/ERROR/CLAUDE_ERROR, guest_count, error_reason) |
| `gateway` | Gateways PMS (pms_type_id, auth_config_json con credenciales OAuth, active) |
| `opera_cloud_data` | Datos sincronizados de Opera Cloud (reservation_id, external_system_code, sync_timestamp) |
| `hotel_to_polling_config` | Config de polling por hotel (poll_interval_seconds, last_poll_time) |
| `guesty_webhook_log` | Webhooks Guesty (event_type, payload_json, processed_status) |
| `infor_webhook_log` | Webhooks Infor |
| `partner` | Partners (PMS) por operador/ubicación (pms_type_id, gateway_id) |

## 7. Integraciones

### Feign Clients
| Cliente | Servicio | URL dev | Propósito |
|---------|----------|---------|-----------|
| IFirebaseFeignService | ms-firebase-service | localhost:9010 | Fallback HTTP para MQPosting/MQTicket (`/firebase/process-ticket`, `/firebase/process-posting`) |

### RabbitMQ
- **Exchange:** `direct.amq.corepark` · **Host dev:** localhost:5671 (SSL)
- **MQPosting** (routing key: `pms.posting.queue`): resultado de post-to-room → consumer: ms-firebase-service
- **MQTicket EDIT_HOTEL_INFO/EDIT_TICKET_INFO** (routing key: `pms.firebase.queue`): edición de info de huésped → consumer: ms-firebase-service
- **Fallback:** AmqpException → `IFirebaseFeignService.processPostingViaHttp()`

### AWS SQS
- **Cola dev:** `hotel-responses-dev.fifo` · **Cola prod:** `hotel-responses.fifo`
- **Región:** us-east-2
- **Propósito:** Eventos Opera On-Premise (checkin, checkout, modification)
- **Mode:** FIFO, ON_SUCCESS ack, 10 mensajes concurrentes

### Claude AI (Anthropic)
- **Modelo:** `claude-sonnet-4-6`
- **Endpoint:** `https://api.anthropic.com/v1/messages`
- **Max tokens:** 32,000 · **Timeout:** 180s · **Reintentos:** 2 con backoff exponencial (1s inicial)
- **API version:** `2023-06-01`
- **Formatos soportados:** PDF (Base64), CSV, TXT (texto plano); XLS/XLSX (legacy parsers)

### Sistemas PMS externos
| PMS | Protocolo | Auth | Notas |
|-----|-----------|------|-------|
| Opera Cloud (OHIP) | REST/JSON | OAuth 2.0 (OCIM: client-credentials; SSD: password grant) | Polling activo, Business Events |
| Opera On-Premise | Email IMAP + SQS | - | Email `operaarrivals@corepark.com`, SQS FIFO |
| StayNTouch | REST/JSON + Webhook | - | Post-to-room, post-to-account |
| Guesty | REST/JSON + Webhook | HMAC Svix SHA256 | Validación de firma obligatoria |
| Infor | REST/JSON + Webhook SOAP | - | OTA_HotelResNotif, JAXB marshalling |

### Gmail (IMAP)
- **Cuenta:** `operaarrivals@corepark.com`
- **Protocolo:** IMAP SSL (imap.gmail.com:993)
- **Dev:** read-only (no marca como leído) · **Prod:** read-write (marca como SEEN)

### Thread pools
| Pool | Threads | Propósito |
|------|---------|-----------|
| autopostingTaskExecutor | 1-2 (queue 25) | Autoposting crítico |
| ohipPollingTaskScheduler | 2 | OHIP polling |
| Scheduler default | 2 | Webhooks, tasks generales |

## 8. Configuración por entorno
| Propiedad | Dev | Prod |
|-----------|-----|------|
| port | 9005 | 80 |
| DB | localhost:5432/coreparkdev | RDS prod |
| RabbitMQ | localhost:5671 | AWS MQ prod |
| SQS Queue | hotel-responses-dev.fifo | hotel-responses.fifo |
| Gmail mode | read-only | read-write |
| Firebase feign | localhost:9010 | ms-firebase-service.internal-corepark.com |
| Arrivals operator/location/partner | 8/1/1 | 274/1/1 |
| CLUSTER_NAME | (vacío) | cluster-2b (desactiva schedulers) |

Variables de entorno (K8s): `NAMESPACE`, `CLUSTER_NAME`

## 9. Cómo correr localmente

**Prerequisitos:**
- Java 17, Maven, PostgreSQL local (coreparkdev), RabbitMQ local
- Credenciales AWS para SQS (en `~/.aws/credentials` o env vars)
- API key de Claude en properties
- Gmail app password (opcional, solo si procesas emails)

**Build y run:**
```bash
cd /Users/israel/Dev/Back-End/ms-pms-service
mvn clean install -DskipTests
java -jar target/PMSService-1.0.jar --spring.profiles.active=dev
```

**Deshabilitar schedulers en dev (evitar polling OHIP y autoposting):**
```bash
java -DCLUSTER_NAME=cluster-2b -jar target/PMSService-1.0.jar --spring.profiles.active=dev
```

**Deshabilitar SQS en dev:**
Agregar `aws.sqs.enabled=false` en application-dev.yml

**Verificar:**
```bash
curl http://localhost:9005/health
# {"status":"UP","timestamp":...}
```
