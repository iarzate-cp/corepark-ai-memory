---
name: Documentación técnica ms-backoffice-service
description: Documentación detallada del microservicio de administración y configuración de CorePark (puerto 9004)
type: project
originSessionId: fc4c993d-0479-4108-8ad1-e63c64662c52
---
# ms-backoffice-service

## 1. Propósito y contexto
Panel de administración central de CorePark. Proporciona la API para configurar y gestionar toda la plataforma: operadores, ubicaciones, empleados, tarifas, integraciones de pago, perfiles de huéspedes, configuración de estaciones, catálogos, agregadores de parking y sistemas PMS. También escucha eventos de auditoría y monitoreo vía RabbitMQ, enviando alertas a Slack.

Usado por la interfaz web de backoffice, la valet app (para configuración), y otros microservicios.

## 2. Tech stack
- **Java 17 · Spring Boot 3.5.14 · Spring Cloud 2025.0.2 · Maven**
- **Artifact:** `com.corepark.backoffice:BackofficeService:1.0`
- **Puerto dev:** 9004 · **Puerto prod:** 80 · **Puerto CI:** 8080
- **Spring Boot AMQP, JDBC, Cache, Web, Validation**
- **Caffeine** (caché en memoria) · **Firebase Admin SDK 9.8.0**
- **AWS SDK S3 2.38.7** · **Apache POI 5.5.1** (Excel import)
- **Slack API Client 1.27.3** · **PostgreSQL 42.7.11**
- **Springdoc OpenAPI 2.8.13**

## 3. Arquitectura y patrones
Capas: **Controller → Service → DAO (JDBC directo, sin ORM)**

Paquetes principales:
```
com.corepark
├── backoffice/
│   ├── catalogue/        (catálogos: vehículos, países, tipos)
│   ├── location/         (CRUD de ubicaciones, partners, request points)
│   ├── guestprofile/     (perfiles de huéspedes + bulk import Excel)
│   ├── station/          (estaciones v1 API)
│   ├── rate/             (tarifas de estacionamiento)
│   ├── payments/         (configuración de pasarelas de pago)
│   ├── evcharging/       (config de carga EV)
│   ├── locationsettings/ (SMS templates, términos, tips)
│   ├── vehiclecatalog/   (catálogo NHTSA)
│   ├── pushmessage/      (mensajes push)
│   ├── survey/           (encuestas)
│   └── common/           (S3Service, FirebaseDao, caché, utils)
├── crm/
│   ├── controller/       (CRMController, PMSController, CatalogController)
│   ├── aggregators/      (OCRA, SpotHero, ParkChirp connections)
│   ├── pms/              (conexiones PMS)
│   ├── mq/               (SystemEventsListener, GatewayTracerListener)
│   └── settings/         (configuraciones CRM)
└── common/
    ├── cache/            (CacheService + CacheConfig Caffeine)
    ├── config/           (OpenApiConfig, CiMockConfig)
    └── pagination/
```

Patrones: Controller-Service-DAO, Caffeine @Cacheable/@CacheEvict, RabbitMQ listeners para monitoreo, evento de carga inicial al startup.

## 4. Endpoints
**Total: 248 endpoints** (verificado con grep sobre src/main, excluyendo Feign interfaces). Es el MS con mayor superficie de API — 60+ controllers.

### Resumen de controllers (conteo por archivo)
| Controller | Endpoints | Controller | Endpoints |
|------------|-----------|------------|-----------|
| LocationController | 21 | EmployeeCenterController | 9 |
| CRMController | 16 | WindcaveController | 8 |
| SMSTemplateController | 10 | SequencesController | 8 |
| PaymentsController | 9 | StationController | 7 |
| PmsConnectionController | 7 | ParkingLayoutController | 7 |
| PmsArrivalsFileController | 6 | LocationSettingsController | 6 |
| GuestProfilesController | 6 | ValetAppSettingsController | 5 |
| SpotChargerController | 5 | RateController | 5 |
| PMSController | 5 | OcraConnectionsController | 5 |
| CompensationsController | 5 | ChargerController | 5 |
| VehicleModelController | 4 | ValidationController | 4 |
| TaxController | 4 | StationTicketSequenceController | 4 |
| RulesController | 4 | PhotosController | 4 |
| PhotoReelsController | 4 | PhoneNumberController | 4 |
| GuestProfileTagsController | 4 | FreedompayController | 4 |
| CarProfilesController | 4 | VehicleMakeController | 3 |
| ReservationsController | 3 | GuestPageController | 3+2 |
| EvChargingConfigurationController | 3 | CatalogueController | 3 |
| CatalogController | 3 | WhiteLabelController | 2 |
| VehicleCatalogController | 2 | ValetAppController | 2+1 |
| SurveyController | 2 | SquareController | 2 |
| ScreenConfigAdminController | 2 | PMSController | 2 |
| LodgingController | 2 | DamageConfigurationController | 2 |
| CrmCommonController | 2 | VehicleCatalogBulkController | 1 |
| StationOverviewController | 1 | ScreenConfigController | 1 |
| PushMessageController | 1 | DeviceAuditInformationController | 1 |
| CommonController | 1 | ChargerTypeController | 1 |
| BulkGuestProfileImportController | 1 | | |

Los controllers con conteo doble (ej. GuestPageController 3+2) son clases en paquetes distintos (backoffice/ y crm/).

### Controllers con detalle de endpoints

### CatalogueController (`/catalogue`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/catalogue/employee` | Catálogo para empleado |
| GET | `/catalogue/init` | Inicializa todos los catálogos (cacheados) |
| POST | `/catalogue/rate` | Catálogo de tasas |

### LocationController (`/location`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| PUT | `/location/new-location` | Crear nueva ubicación |
| PUT | `/location/update-location` | Actualizar ubicación |
| POST | `/location/get-location-summary` | Resumen de ubicación |
| POST | `/location/get-location-detail` | Detalle completo |
| PUT | `/location/new-partner` | Crear socio (PMS partner) |
| PUT | `/location/update-partner` | Actualizar socio |
| POST | `/location/get-partner-summary` | Resumen de socio |
| POST | `/location/get-partner-detail` | Detalle de socio |
| PUT | `/location/new-request-point` | Crear punto de solicitud (donde guest pide el auto) |
| PUT | `/location/update-request-point` | Actualizar punto |
| POST | `/location/get-request-point-summary` | Resumen de punto |
| POST | `/location/get-request-point-detail` | Detalle de punto |
| POST | `/location/get-partners-catalogue` | Catálogo de socios |
| POST | `/location/grant-access` | Dar acceso a empleado en ubicación |
| POST | `/location/revoke-access` | Revocar acceso |
| POST | `/location/access-catalogue` | Acceso de empleado a ubicación |
| GET | `/location/deal-structures` | Estructuras de acuerdos |
| GET | `/location/location-configurations` | Config de ubicación (Headers: operator-id, location-id) |
| PUT | `/location/location-configurations` | Actualizar config (ValetAppConfigurationRequest) |
| PUT | `/location/update-vehicle-request-capacity` | Actualizar capacidad de solicitudes |
| GET | `/location/get-vehicle-request-capacity` | Obtener capacidad |

### GuestProfilesController (`/guest-profiles`)
| Método | Ruta | Descripción | Headers |
|--------|------|-------------|---------|
| POST | `/guest-profiles` | Crear perfil | operator-id, location-id |
| GET | `/guest-profiles` | Listar todos | operator-id, location-id |
| GET | `/guest-profiles/{id}` | Obtener por ID | - |
| POST | `/guest-profiles/search` | Buscar con filtros | - |
| PUT | `/guest-profiles/{id}` | Actualizar | - |
| DELETE | `/guest-profiles/{id}` | Eliminar | - |
| POST | `/guest-profiles/bulk-import/{locationId}` | Importar masivo desde XLSX (multipart/form-data) | operator-id |

### StationController (`/v1/station`)
| Método | Ruta | Descripción | Headers |
|--------|------|-------------|---------|
| POST | `/v1/station` | Crear estación | Operator-Id, Location-Id |
| GET | `/v1/station` | Listar estaciones | Operator-Id, Location-Id |
| GET | `/v1/station/{uuid}` | Obtener por UUID | Operator-Id, Location-Id |
| PUT | `/v1/station` | Actualizar | Operator-Id, Location-Id |
| DELETE | `/v1/station/{uuid}` | Eliminar | Operator-Id, Location-Id |
| POST | `/v1/station/{uuid}/reactivate` | Reactivar | Operator-Id, Location-Id |
| POST | `/v1/station/valet-app/assignment` | Asignar a dispositivo Android | Operator-Id, Location-Id |

### LocationSettingsController (`/location-settings`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| PUT | `/location-settings/sms/new-sms-configuration` | Crear config SMS |
| POST | `/location-settings/sms/get-sms-configuration` | Obtener config SMS |
| PUT | `/location-settings/sms/new-terms-conditions` | Crear términos y condiciones |
| POST | `/location-settings/sms/get-terms-conditions` | Obtener términos |
| POST | `/location-settings/tip/update-tip-configuration` | Actualizar config de propinas |
| POST | `/location-settings/tip/get-tip-configuration` | Obtener config de propinas |

### EvChargingConfigController (`/v1/config-parameters/ev-charging`)
| Método | Ruta | Headers |
|--------|------|---------|
| GET | `/v1/config-parameters/ev-charging` | Operator-Id, Location-Id |
| GET | `/v1/config-parameters/ev-charging/settings` | Operator-Id, Location-Id |
| PATCH | `/v1/config-parameters/ev-charging` | Operator-Id, Location-Id |

### PaymentController (`/payment`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/payment/credentials` | Estado de pasarelas (Header: Operator-Id) |
| POST | `/payment/credentials/connection` | Conectar ubicación a pasarela |
| PUT | `/payment/credentials/connection` | Editar conexión |
| DELETE | `/payment/credentials/connection/{uuid}` | Desconectar |
| GET | `/payment/locations-connection` | Conexiones por ubicación |
| GET | `/payment/location-detail` | Detalle de configuración |
| GET | `/payment/valet-app/configuration` | Config para Valet App |
| GET | `/payment/guest-page/configuration` | Config para guest page |
| POST | `/payment/disconnect-location` | Desconectar ubicación |
| PUT | `/payment/square/connect-location` | Conectar a Square |
| POST | `/payment/square/update-location` | Actualizar conexión Square |

### RateController (`/rate`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| PUT | `/rate/new-parking-rate` | Crear tarifa |
| POST | `/rate/get-parking-rate` | Obtener tarifas |
| PUT | `/rate/update-parking-rate` | Actualizar tarifa |
| PUT | `/rate/delete-parking-rate` | Eliminar tarifa |
| POST | `/rate/get-rates-catalogue` | Catálogo de tasas |

### CRMController (`/crm`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/crm/health` | Health check |
| POST | `/crm/prospects-clients` | Operadores pendientes |
| POST | `/crm/to-decline-request` | Rechazar solicitud |
| POST | `/crm/to-ontrial-operator` | Pasar a prueba |
| POST | `/crm/operator-request` | Crear solicitud de operador |
| PUT | `/crm/operator-request` | Editar solicitud |
| GET | `/crm/operator-request/{id}` | Info de operador |
| POST | `/crm/add-comment` | Añadir comentario |
| GET | `/crm/operator-comments/{id}` | Comentarios de operador |
| GET | `/crm/operator-details/{id}` | Detalles de operador |
| PUT | `/crm/to-inactive-operator` | Inactivar operador |
| PUT | `/crm/to-operating-operator` | Pasar a operativo |
| GET | `/crm/locations` | Ubicaciones (Header: Operator-Id) |
| PUT | `/crm/uuid` | Actualizar UUID |
| POST | `/crm/otp` | Generar OTP |
| GET | `/crm/hash` | Hash value |

### OCRA Connections (`/crm/aggregators/connections/ocra`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/crm/aggregators/connections/ocra/facilities` | Listar instalaciones |
| POST | `/crm/aggregators/connections/ocra/facilities/sync` | Sincronizar instalaciones (invalida caché) |
| GET | `/crm/aggregators/connections/ocra/links` | Listar conexiones |
| POST | `/crm/aggregators/connections/ocra/links` | Crear conexión |
| DELETE | `/crm/aggregators/connections/ocra/links/{id}` | Eliminar conexión |

### ReservationsController (`/crm/aggregators/reservations`)
| Método | Ruta | Query Params |
|--------|------|-------------|
| GET | `/crm/aggregators/reservations/configured-aggregators` | parkingLocationId |
| GET | `/crm/aggregators/reservations` | aggregator, dateFrom, dateTo, barcode, page, size |
| GET | `/crm/aggregators/reservations/export` | aggregator, dateFrom, dateTo, barcode (CSV) |

## 5. Lógica de negocio principal

### Gestión de catálogos (caché Caffeine)
`/catalogue/init` retorna idiomas, tipos de ubicación, países, perfiles, diseños, tasas. Cacheados en Caffeine (2000 entradas max), invalidados manualmente. CacheService centralizado con métodos tipados.

Caché clave: `OCRA_FACILITIES` (instalaciones OCRA), `LOCATION_TIMEZONE` (timezone por operator+location).

### Bulk import de perfiles de huéspedes
`POST /guest-profiles/bulk-import/{locationId}` acepta XLSX. `BulkGuestProfileImportService` parsea con Apache POI, valida datos, inserta en lote. Retorna resumen con éxitos/errores/duplicados.

### Creación de ubicaciones
Valida alias único, timezone IANA válido (via Google Maps API para coordenadas → timezone), luego inserta en transacción.

### Monitoreo via RabbitMQ
- **`system-monitoring` queue** → `SystemEventsListener`: recibe errores HTTP, envía alerta a Slack (#dev-backend-monitoring en dev)
- **`gateway-tracer` queue** → `GatewayTracerListener`: registra auditoría de acceso de la valet app

### Integración PMS
Configuración de conexiones con Opera, StayNTouch, Guesty, Infor por operador/ubicación.

## 6. Base de datos
**Dev:** `jdbc:postgresql://127.0.0.1:5432/coreparkdev` · user: `dewsdbcp` · pass: `xxv87htXui8iHm`
**Prod:** RDS us-east-2 `corepark` · user: `prwsdbcp`
**Pool Hikari:** min 2, max 5, idle 10min

Tablas principales:
| Tabla | Descripción |
|-------|-------------|
| `company.parking_location` | Ubicaciones (alias, timezone, config) |
| `company.operator_company` | Operadores |
| `company.parking_location_worker` | Acceso empleado-ubicación |
| `backoffice.parking_rate` | Tarifas (flat, hourly_fixed, hourly_variable, temporary, overnight) |
| `backoffice.guest_profile` | Perfiles de huéspedes |
| `backoffice.station` | Estaciones (uuid PK, deleted_at) |
| `backoffice.payment_gateway_configuration` | Config de pasarelas (Square, Freedompay, Windcave) |
| `backoffice.sms_configuration` | Plantillas SMS por ubicación |
| `backoffice.location_setting` | Config de propinas |
| `crm.device_audit_information` | Auditoría de acceso valet app |
| `crm.ocra_connection_link` | Conexiones con OCRA |
| `crm.pms_connection` | Configuraciones de PMS |

## 7. Integraciones

### Firebase Realtime Database
- **Propósito:** Almacenamiento en tiempo real, push notifications
- **Credencial:** `corepark-services-firebase-adminsdk.json`
- **URL dev:** `https://corepark-services-dev.firebaseio.com`
- **URL prod:** `https://corepark-services.firebaseio.com`

### AWS S3
- **Bucket `damage-image-content`** (us-east-2): imágenes de daños de vehículos
- **Bucket `operators-media`** (us-east-1): media de operadores
- **Cliente:** AWS SDK v2 con 2 instancias S3Client configuradas (una por región)

### Google Maps API
- **Propósito:** Obtener timezone de coordenadas al crear ubicación
- **Endpoint:** `https://maps.googleapis.com/maps/api/timezone/json`

### Square Payment Gateway
- **Propósito:** Procesamiento de pagos por tarjeta
- **Client ID dev:** sq0idp-zmQWvyLTpsTozOiLf-9j1g · **Prod:** sq0idp-Nfa7m3PwnXSbzAZ8spWd_A

### RabbitMQ
- **Exchange:** `direct.amq.corepark`
- **Host dev:** `127.0.0.1:5671` (SSL) · **Prod:** AWS MQ
- **Queue `system-monitoring`** → escucha errores HTTP → alerta Slack
- **Queue `gateway-tracer`** → auditoría de dispositivos

### Slack API
- **Dev channel:** `#dev-backend-monitoring`
- **Prod channels:** `#web-backend-monitoring`, `#mobile-backend-monitoring`, `#customer-bugs`
- **Payload:** error HTTP, latencia, URL, request, response, info de operador

### Agregadores de Parking
| Agregador | URL dev | URL prod |
|-----------|---------|---------|
| OCRA | https://partners.stage.getocra.com | https://partners.getocra.com |
| SpotHero | https://spothero.com | https://spothero.com |
| ParkChirp | https://api.parkcheep.com/external/v1 | https://api.parkchirp.com/external/v1 |

## 8. Configuración por entorno
| Propiedad | Dev | Prod |
|-----------|-----|------|
| port | 9004 | 80 |
| DB | localhost:5432/coreparkdev | RDS prod |
| Firebase URL | dev.firebaseio.com | firebaseio.com |
| Square Client ID | sq0idp-zmQ... | sq0idp-Nfa... |
| OCRA URL | stage | prod |
| ParkChirp URL | parkcheep.com | parkchirp.com |
| S3 damage bucket | damage-image-content | damage-image-content-prod |
| Slack channels | #dev-backend-monitoring | #web/mobile-backend-monitoring |

Variables de entorno (K8s): `NAMESPACE`, `CLUSTER_NAME`

## 9. Cómo correr localmente

**Prerequisitos:**
- Java 17, Maven, PostgreSQL local (coreparkdev), RabbitMQ local
- Archivo `corepark-services-firebase-adminsdk.json` en classpath
- AWS credentials para S3 (en `~/.aws/credentials` o env vars)

**Build y run:**
```bash
cd /Users/israel/Dev/Back-End/ms-backoffice-service
mvn clean package -DskipTests
java -jar target/BackofficeService-1.0.jar --spring.profiles.active=dev
# O con variables de entorno explícitas
export AWS_ACCESS_KEY_ID="<REDACTED — véase AWS IAM console / 1Password>"
export AWS_SECRET_ACCESS_KEY="..."
java -jar target/BackofficeService-1.0.jar --spring.profiles.active=dev
```

**Verificar:**
```bash
curl http://localhost:9004/crm/health
# {"status":"UP"}
```

**Nota:** Sin RabbitMQ corriendo, los listeners de monitoreo fallarán al startup. Sin Firebase JSON, los endpoints que usan Firebase fallarán en runtime.
