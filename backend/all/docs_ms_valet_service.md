---
name: Documentación técnica ms-valet-service
description: Documentación detallada del microservicio central de operaciones de valet parking (puerto 9000)
type: project
originSessionId: fc4c993d-0479-4108-8ad1-e63c64662c52
---
# ms-valet-service

## 1. Propósito y contexto
Núcleo operativo de CorePark. Gestiona el ciclo completo de vida de un vehículo en valet: check-in, estacionamiento, solicitud, entrega y check-out. También maneja control de efectivo, cupones, estacionamiento nocturno, carga EV, tracking de dispositivos e integración con agregadores de parking y Opera PMS.

Actúa como hub de orquestación entre Firebase (notificaciones en tiempo real), ms-payments-service (pagos), ms-pms-service (PMS hotelero), ms-notifications-service (SMS) y ms-firebase-service (push).

## 2. Tech stack
- **Java 17 · Spring Boot 3.5.14 · Spring Cloud 2025.0.2 · Maven**
- **Artifact:** `com.corepark.valet:ValetService:1.0`
- **Puerto dev:** 9000 · **Puerto prod:** 80
- **Spring Boot AMQP, JDBC, Web, Validation**
- **Spring Cloud OpenFeign + Load Balancer**
- **PostgreSQL 42.7.11 · Springdoc OpenAPI 2.8.13**

## 3. Arquitectura y patrones
Capas: **Controller → Service (interfaz + impl) → DAO (JDBC directo)**

Paquetes principales:
```
com.corepark.valet
├── valet/          (ValetController — operaciones principales)
├── tickets/        (TicketsController — solicitud de auto)
├── payments/       (PaymentController + PaymentFeignService)
├── cashcontrol/    (CashControlController)
├── evcharging/     (EvChargingController)
├── coupon/         (CouponController)
├── scan/           (ScanController — QR/barcode)
├── validations/    (ValidationsController)
├── transactions/   (TransactionsController)
├── overnight/      (OvernightController)
├── parkingaggregator/ (OCRA, SpotHero, ParkChirp)
├── stackparking/   (estacionamiento automático)
├── opera/          (OperaController — PMS)
├── device/         (AndroidDeviceTrackingController)
├── notifications/  (RequestNotificationTrackController)
├── rabbitmq/       (IRabbitService — publica eventos)
├── feign/          (clientes HTTP a otros MS)
├── commons/        (ResponseBean, enums, validators)
└── config/         (JacksonConfig, OpenApiConfig, RestClientConfig)
```

Patrones: Controller-Service-DAO, NamedParameterJdbcTemplate, DTOs con @Valid, RabbitMQ con fallback HTTP a Firebase.

## 4. Endpoints
**Total: 75 endpoints** (verificado con grep sobre src/main, excluyendo Feign interfaces)

### ValetController (`/`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/health` | Health check |
| POST | `/registry` | Registrar empleado |
| POST | `/clock-in` | Empleado inicia turno |
| POST | `/clock-out` | Empleado termina turno |
| POST | `/employee-details` | Detalles de empleado (phoneCode, phoneNumber) |
| PUT | `/assing-pincode` | Asignar PIN a empleado en ubicación |
| GET | `/catalogs/country-codes` | Códigos de país |
| GET | `/v1/check-in/eligibility` | Valida precondiciones de check-in (Headers: Operator-Id, Location-Id; Query: ticket) |
| POST | `/v2/check-in` | Hacer check-in (body: ticket, rate, guest, vehicle, overnight) |
| PUT | `/ticket-info` | Editar info de ticket ya marcado |
| GET | `/v1/check-out/ticket/{ticket}/eligibility` | Valida si puede hacer checkout |
| POST | `/v2/check-out` | Hacer check-out (lista de tickets con amountPaid, paymentMethod) |
| POST | `/attend-request-car` | Valet comienza a atender solicitud |
| POST | `/complete-request-car` | Valet completa atención |
| POST | `/accept-cancellation-request` | Aceptar cancelación |
| POST | `/pickup` | Recoger vehículo |
| POST | `/park` | Asignar espacio y estacionar (row, spot, parkingAreaId) |
| POST | `/relocate` | Mover vehículo a otro espacio |
| POST | `/mark-en-route` | Valet en camino con vehículo |
| POST | `/mark-hold` | Marcar en espera |
| PUT | `/change-rate` | Cambiar tarifa |
| POST | `/get-rate-info-total-collect` | Calcular total a cobrar |
| GET | `/vin-information/{vin}` | Decodificar VIN (make, model, year) |
| POST | `/key-not-retrieved` | Registrar llaves no recuperadas |
| POST | `/compensation` | Aplicar compensación |
| POST | `/ticket` | Obtener estado del ticket |
| POST | `/v1/transfer-tickets` | Transferir tickets entre estaciones |
| POST | `/resend-sms` / `/resend-sms-v2` / `/v3/resend-sms` | Reenviar SMS |

### TicketsController (`/tickets/`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/tickets/checkouts/latest` | Últimos tickets en checkout |
| POST | `/tickets/reinstate` | Reinstalar ticket (overnight/reutilización) |
| POST | `/tickets/request` | Guest solicita su vehículo |
| PUT | `/tickets/cancel-request` | Cancelar solicitud de vehículo |

### ScanController (`/scan/`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/scan/resolve` | Resolver código QR/barcode escaneado (ticket, guest profile, agregador) |

### PaymentController (`/payment`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/payment/v2/add` | Registrar prepago para tickets |

### CashControlController (`/cash-control`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/cash-control/get-activity-log` | Registro de movimientos de efectivo |
| POST | `/cash-control/get-cash-register-amount` | Saldo actual de caja |
| POST | `/cash-control/reconcile` | Reconciliar efectivo |
| POST | `/cash-control/get-reconciliation-detail` | Detalle de última reconciliación |
| POST | `/cash-control/withdrawal` | Retirar efectivo de caja |

### EvChargingController (`/v1/ev-charging/`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| PUT | `/v1/ev-charging/ticket/{ticket}/registration` | Registrar vehículo EV |
| POST | `/v1/ev-charging/ticket/{ticket}/plug-in` | Conectar a cargador |
| POST | `/v1/ev-charging/ticket/{ticket}/plug-out` | Desconectar de cargador |
| POST | `/v1/ev-charging/ticket/{ticket}/switch-charger` | Cambiar de cargador |
| POST | `/v1/ev-charging/ticket/{ticket}/enqueue` | Encolar para carga |
| POST | `/v1/ev-charging/ticket/{ticket}/estimate-charge-time` | Estimar tiempo de carga |
| GET | `/v1/ev-charging/tickets` | Tickets con sesiones EV activas |

### CouponController (`/v1/coupon`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/v1/coupon/scan/{scanCode}` | Leer metadata de cupón (read-only) |
| POST | `/v1/coupon/application` | Aplicar cupón a ticket |
| DELETE | `/v1/coupon/application/{ticket}/{scanCode}` | Remover cupón de ticket |
| GET | `/v1/coupon/ticket/{ticket}/scan-codes` | Cupones aplicados a ticket |

### OvernightController (`/overnight`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/overnight/temporal-check-out` | Checkout temporal (va al hotel) |
| POST | `/overnight/re-check-in` | Re-check-in del día siguiente |
| PUT | `/overnight/assign-room` | Asignar habitación a ticket |
| POST | `/overnight/check-if-checkout-can-be-made` | Validar si puede hacer checkout |

### TransactionsController (`/transactions`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/transactions/recents` | Últimas transacciones |
| POST | `/transactions/detail` | Detalle de transacción |
| POST | `/transactions/refund` | Procesar reembolso |

## 5. Lógica de negocio principal

### Check-in flow
1. **Eligibility** (`/v1/check-in/eligibility`): valida daily report iniciado, rates activos, ticket no en uso, política de reutilización (120s dev / 86400s prod)
2. **Check-in** (`/v2/check-in`): inserta `company.parking_service`, publica MQTicket CHECK_IN vía RabbitMQ (fallback HTTP)

### Check-out flow
1. **Eligibility** (`/v1/check-out/ticket/{ticket}/eligibility`): valida ticket activo, pagos, validaciones
2. **Check-out** (`/v2/check-out`): procesa pago (PaymentFeignService), actualiza `check_out = NOW()`, publica MQTicket COMPLETE

### Valet attend/park/deliver flow
`check-in → park → (request car) → attend-request → mark-en-route → complete-request → check-out`
Cada transición publica MQTicket con status correspondiente vía RabbitMQ

### Cash control
Activity log → reconcile (compara declared vs calculated) → withdrawal; diferencias registradas como overage/shortage

### EV Charging
registration → plug-in → (estimate-charge-time) → plug-out; cron diario a medianoche limpia sesiones expiradas (24h)

### Coupon application
scan (read-only) → application (valida voucher activo, ticket sin validación/compensación activa) → checkout marca como usado

## 6. Base de datos
**Dev:** `jdbc:postgresql://localhost:5432/coreparkdev` · user: `dewsdbcp` · pass: `xxv87htXui8iHm`
**Prod:** RDS us-east-2 `corepark` · user: `prwsdbcp`
**Pool Hikari:** min 2, max 5, idle 10min

Tablas principales (esquema `company`):
| Tabla | Descripción |
|-------|-------------|
| `company.parking_service` | Registro principal: ticket (PK), check_in, check_out, rate, guest info, vehicle info, spot asignado, estado, overnight info |
| `company.parking_service_info` | Historial de eventos por ticket (service_status_id, valet_credential_id, row, spot) |
| `company.employee` | Empleados valet |
| `company.parking_location_worker` | Mapeo empleado-ubicación (pin_code, comp_allowed) |
| `company.shift` | Turnos (clock_in, clock_out) |
| `company.ticket_ev_data` | Datos EV por ticket (battery_capacity, current_level, charge_target) |
| `company.ticket_ev_charging_session` | Sesiones de carga EV activas |
| `company.ticket_coupon_application` | Cupones aplicados (status active/removed) |
| `company.ticket_flight_info` | Info de vuelos asociados a ticket |
| `company.cash_register_activity` | Movimientos de efectivo |
| `company.cash_register_reconciliation` | Reconciliaciones |
| `company.parking_area` | Áreas de estacionamiento |
| `company.parking_location` | Ubicaciones (timezone, daily_report_started) |

## 7. Integraciones

### Feign Clients (HTTP)
| Cliente | Servicio | URL dev | Propósito |
|---------|----------|---------|-----------|
| PaymentFeignService | ms-payments-service | localhost:9001 | Reembolsos (`POST /common/refund-transaction`) |
| IPmsFeignService | ms-pms-service | localhost:9005 | Opera posting (`POST /opera/legacy/guest-posting`) |
| IFirebaseFeignService | ms-firebase-service | localhost:9010 | Push notifications + EV events (múltiples endpoints) |
| INotificationsFeignService | ms-notifications-service | localhost:9006 | SMS (`POST /sms/send-single-sms`, `/sms/send-message-template`) |
| IBackofficeFeignService | ms-backoffice-service | localhost:9004 | Config EV charging |

Timeouts: connectTimeout 60s, readTimeout 62s

### RabbitMQ
- **Exchange:** `direct.amq.corepark`
- **Host dev:** `127.0.0.1:5671` (SSL) · **Prod:** AWS MQ
- **Mensajes publicados (MQTicket):** CHECK_IN, ATTEND, EN_ROUTE, HOLD, PARK, PARK_RELOCATE, REQUEST, COMPLETE, TRANSFER
- **Fallback:** Si AmqpException → Feign HTTP a Firebase

### APIs externas
| API | Propósito | URL dev |
|-----|-----------|---------|
| VPIC (NHTSA) | Decodificación VIN | https://vpic.nhtsa.dot.gov/api/vehicles/DecodeVin/ |
| ParkNet | SSL parking | https://testonlineservices.parknet.net |
| AWS URL Shortener | Acortar URLs | Lambda en us-east-2 |
| OCRA | Agregador parking | https://partners.stage.getocra.com |
| SpotHero | Agregador parking | https://spothero.com |
| ParkChirp | Agregador parking | https://api.parkcheep.com/external/v1 |
| EVHA | EV charging | api-key en properties |
| Way | EV charging | api-key en properties |

## 8. Configuración por entorno
| Propiedad | Dev (localhost) | Dev (AWS) | Prod |
|-----------|----------------|-----------|------|
| port | 9000 | 80 | 80 |
| DB URL | localhost:5432/coreparkdev | dev RDS | prod RDS |
| RabbitMQ | localhost:5671 | AWS MQ dev | AWS MQ prod |
| time-without-allowing-check-in | 120s | 300s | 86400s |
| OCRA URL | stage | stage | prod |
| EVHA/Way keys | sk_dev_* | sk_dev_* | sk_prod_* |

Variables de entorno (K8s): `NAMESPACE`, `CLUSTER_NAME`

## 9. Cómo correr localmente

**Prerequisitos:**
- Java 17, Maven, PostgreSQL local (coreparkdev), RabbitMQ local

**Base de datos:**
```bash
createdb -U postgres coreparkdev
psql -U postgres coreparkdev < schema.sql  # obtener dump del DBA
```

**RabbitMQ (Docker):**
```bash
docker run -d --name rabbitmq -p 5671:5671 -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=rabbit-admin \
  -e RABBITMQ_DEFAULT_PASS='qu3.n0-v3rg#.3nt1EnD32024$' \
  rabbitmq:latest
```

**Build y run:**
```bash
cd /Users/israel/Dev/Back-End/ms-valet-service
mvn clean package -DskipTests
java -jar target/ValetService-1.0.jar --spring.profiles.active=dev
# O con Maven:
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"
```

**Verificar:**
```bash
curl http://localhost:9000/health
# {"status":"UP","timestamp":...}
```

**Otros MS requeridos para funcionalidad completa:** ms-payments-service (:9001), ms-pms-service (:9005), ms-firebase-service (:9010), ms-notifications-service (:9006), ms-backoffice-service (:9004)
