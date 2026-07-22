---
name: Documentación técnica microservice-reports
description: Documentación detallada del microservicio de generación y distribución de reportes (puerto 9002)
type: project
originSessionId: fc4c993d-0479-4108-8ad1-e63c64662c52
---
# microservice-reports

## 1. Propósito y contexto
Motor de reportes de CorePark. Genera, programa y distribuye reportes operacionales en múltiples formatos (CSV, Excel/XLSX, PDF) para operadores de estacionamiento. Soporta 34 tipos de reportes específicos del dominio, programación recurrente con timezone awareness, y distribución automática por email.

Se integra principalmente con **ms-notifications-service** para el envío de emails con adjuntos.

## 2. Tech stack
- **Java 17 · Spring Boot 3.5.14 · Spring Cloud 2025.0.2 · Maven**
- **Artifact:** `com.corepark:ReportsService:1.0`
- **Puerto dev:** 9002 · **Puerto prod:** 80
- **Apache POI 5.4.0** (Excel), **PDFBox 2.0.24** (PDF), **Commons CSV 1.8**
- **Spring Cloud OpenFeign** (cliente HTTP a notifications)
- **PostgreSQL 42.7.11 · Springdoc OpenAPI 2.8.13**
- **Lombok 1.18.38**

## 3. Arquitectura y patrones
Capas: **Controller → Service (interfaz + impl) → DAO (JDBC directo)**

Estructura de paquetes:
```
com.corepark.report
├── common/           (beans, config, DAO, exceptions, util)
├── reporchestrator/  (API unificada de gestión de reportes)
├── scheduler/        (programación + AutomatedReportSchedulerService @Scheduled)
├── master/           (reporte master multidimensional)
├── dailyreport/      (cierre diario)
├── overnight/        (reporte nocturno)
├── revenue/          (ingresos)
├── ticketlistreport/ (lista de tickets)
├── inventoryreport/  (inventario)
├── notifications/    (EmailService FeignClient)
└── [+26 módulos de reportes específicos]
```

Patrones: MVC, DAO pattern, @Scheduled para automatización, FeignClient multipart para envío de emails con adjuntos.

## 4. Endpoints
**Total: 87 endpoints** (verificado con grep sobre src/main, excluyendo Feign interfaces)

### ReportController
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/health` | Health check |

### ReportOrchestratorController (`/report-orchestrator`)
| Método | Ruta | Descripción | Headers |
|--------|------|-------------|---------|
| GET | `/report-orchestrator/reports` | Catálogo de reportes disponibles | `operator-id` |
| GET | `/report-orchestrator/intervals` | Catálogo de intervalos | `operator-id` |
| POST | `/report-orchestrator/scheduled` | Crear reporte programado | `operator-id` |
| POST | `/report-orchestrator/automated` | Crear reporte automático | `operator-id` |
| PUT | `/report-orchestrator/scheduled/{uuid}` | Actualizar reporte programado | `operator-id`, `location-id` |
| PUT | `/report-orchestrator/scheduled/{uuid}/emails` | Actualizar emails de reporte | `operator-id`, `location-id` |
| PUT | `/report-orchestrator/automated/{uuid}/emails` | Actualizar emails automatizado | `operator-id`, `location-id` |
| DELETE | `/report-orchestrator/scheduled/{uuid}` | Eliminar reporte programado | `operator-id`, `location-id` |
| DELETE | `/report-orchestrator/automated/{uuid}` | Eliminar reporte automático | `operator-id`, `location-id` |
| POST | `/report-orchestrator/delete` | Eliminar múltiples reportes | `operator-id`, `location-id` |
| POST | `/report-orchestrator/by-operator` | Reportes por operador | `operator-id` |
| POST | `/report-orchestrator/statistics` | Estadísticas de reportes | `operator-id` |
| POST | `/report-orchestrator/filter` | Filtrar reportes | `operator-id` |
| POST | `/report-orchestrator/emails` | Eliminar emails de reportes | `operator-id` |
| POST | `/report-orchestrator/reports/search` | Búsqueda avanzada | `operator-id` |

### SchedulerController (`/scheduler`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| PUT | `/scheduler/new-scheduled-report` | Crear reporte programado (legado) |
| POST | `/scheduler/get-scheduled-report` | Obtener reportes de ubicación |
| PUT | `/scheduler/update-scheduled-report` | Actualizar reporte |
| POST | `/scheduler/delete-scheduled-report` | Desactivar reporte |
| POST | `/scheduler/request-automated-report-run` | Ejecución manual |

### DailyReportController (`/daily-report`, `/daily`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/daily-report/get-summary` | Resumen diario |
| POST | `/daily-report/get-detail` | Detalle diario |
| POST | `/daily-report/get-shifts` | Turnos programados |
| POST | `/daily/start` | Iniciar cierre diario |
| POST | `/daily/is-started` | Verificar si cierre está iniciado |
| POST | `/daily/get-cars-still-parked` | Vehículos todavía estacionados |
| POST | `/daily/end` | Finalizar cierre |
| POST | `/daily/generate` | Generar reporte |
| POST | `/daily/close` | Cerrar reporte |

### MasterController (`/master`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/master/get-master-report` | Reporte master en JSON |
| POST | `/master/get-master-report-pdf` | Exportar master a PDF (byte[]) |

### OvernightController (`/overnight`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/overnight/get-summary` | Resumen nocturno |
| POST | `/overnight/export-overnight` | Exportar a CSV (byte[]) |
| POST | `/overnight/export-overnight-xls` | Exportar a XLSX (byte[]) |

### RevenueController (`/revenue`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/revenue` | Reporte de ingresos |
| POST | `/revenue/export-xlsx` | Exportar a XLSX (byte[]) |

### TicketListReportController (`/ticket-list`)
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/ticket-list/get-ticket-list` | Lista de tickets |
| POST | `/ticket-list/get-ticket-detail` | Detalle de ticket |
| POST | `/ticket-list/export/excel` | Exportar a XLSX |

### InventoryReportController (`/inventory-report`)
| Método | Ruta | Descripción | Headers |
|--------|------|-------------|---------|
| GET | `/inventory-report` | Reporte de inventario | `operator-id`, `location-id` |

**Otros controladores (mismo patrón):** DashboardController, TipsController, CashReconcileController, EmployeeReportController, DeviceAuditController, TheftReportController, CompensatedTicketsReportController, SpotUsageController, TicketTransactionController, TicketDurationController, LaborTipController, EmployeePerformanceReportController, CeoController, HotelChargeController, TimeTrackingController, TrendsController, RevenueAndLaborController, FutureRequestController, GarageController, SmsUssageController, ValidationsController

## 5. Lógica de negocio principal

### Crear reporte programado
`POST /report-orchestrator/scheduled` → valida emails, calcula timezone de la ubicación, calcula `run_from`/`run_to`, inserta en `company.scheduled_report` + `company.scheduled_report_email`. Si es Master Report: inserta también en `company.scheduled_master_report_cfg`.

### Ejecución automática (Scheduler)
`AutomatedReportSchedulerService` corre cada **5 minutos** (`@Scheduled(fixedDelay = 1000 * 60 * 5)`):
1. Consulta `company.scheduled_report` donde `start_at < NOW()`, `NOW() >= next_run_to`, `deactivated_at IS NULL`
2. Para cada reporte: obtiene config + recipients, invoca servicio específico, genera archivo, envía email vía Feign a `ms-notifications-service`
3. Actualiza `executed_at`, calcula `next_run_to` según intervalo

El scheduler es **cluster-aware**: solo se activa en ciertos nodos (controlado por propiedad `CLUSTER_NAME`).

### Daily report close trigger
Cuando se ejecuta `/daily/close`, los reportes automáticos asociados al cierre diario se ejecutan en el siguiente tick del scheduler.

### Generación de reportes (ejemplo Master Report)
Queries múltiples SQL (transacciones, tips, comisiones, impuestos, filtros por rate), construye `MasterReportData`, retorna JSON o genera PDF con PDFBox.

## 6. Base de datos
**Dev:** `jdbc:postgresql://127.0.0.1:5432/coreparkdev` · user: `dewsdbcp` · pass: `xxv87htXui8iHm`
**Prod:** RDS us-east-2 `corepark` · user: `prwsdbcp`
**Pool Hikari:** min 2, max 5, idle 10min

Tablas principales:
| Tabla | Descripción |
|-------|-------------|
| `company.scheduled_report` | Reportes programados (uuid, report_id, interval_id, start_at, next_run_to, deactivated_at, cvs, pdf) |
| `company.scheduled_report_email` | Destinatarios de reportes programados |
| `company.scheduled_master_report_cfg` | Config del Master Report (cc_tips, tax, comps, validations, all_rates) |
| `company.automated_report` | Reportes automáticos (uuid, report_id, file_name_prefix) |
| `company.distribution_email` | Emails de distribución para reportes automáticos |
| `company.automated_report_request` | Solicitudes de ejecución (run_from, run_to, executed_at) |
| `custom.cat_interval` | Catálogo de intervalos (daily, weekly, monthly) |
| `custom.cat_report` | Catálogo de reportes disponibles |

## 7. Integraciones

### Feign Clients
| Cliente | Servicio | URL dev | Endpoint | Propósito |
|---------|----------|---------|----------|-----------|
| EmailService | ms-notifications-service | http://127.0.0.1:9006 | `POST /email/reports/overnight` | Enviar reportes como adjuntos de email (multipart/form-data) |

Timeout: connectTimeout 60s, readTimeout 62s

No tiene RabbitMQ ni integraciones con APIs externas directas.

## 8. Configuración por entorno
| Propiedad | Dev | Prod |
|-----------|-----|------|
| port | 9002 | 80 |
| DB URL | 127.0.0.1:5432/coreparkdev | RDS prod |
| notifications URL | http://127.0.0.1:9006 | http://ms-notifications-service.internal-corepark.com |
| app name | ms-reports-service | ms-reports-service.internal-corepark.com |
| scheduling thread pool | 8 threads | 8 threads |
| cluster-aware scheduling | activo | solo en nodo designado |

Variables de entorno (K8s): `NAMESPACE`, `CLUSTER_NAME`

## 9. Cómo correr localmente

**Prerequisitos:**
- Java 17, Maven, PostgreSQL local (coreparkdev)
- ms-notifications-service corriendo en :9006 (para envío de emails)

**Build y run:**
```bash
cd /Users/israel/Dev/Back-End/microservice-reports
mvn clean package -DskipTests
java -jar target/ReportsService-1.0.jar --spring.profiles.active=dev
# O con Maven:
./mvnw spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"
```

**Verificar:**
```bash
curl http://localhost:9002/health
# {"status":"UP","timestamp":...}
```

**Nota:** Sin ms-notifications-service corriendo, los reportes se generan pero el envío de emails fallará.
