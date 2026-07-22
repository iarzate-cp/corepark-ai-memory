---
name: Microservice communication in the Corepark ecosystem
description: How MS talk to each other — Feign, RestTemplate, RabbitMQ, gateway. What's wired in ms-backoffice-service today
type: reference
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
Backend ecosystem (`/Users/israel/Dev/Back-End/`) has 8+ microservices. Communication mechanisms discovered on 2026-07-01:

## 1. Feign (synchronous HTTP) — the main internal channel

Spring Cloud OpenFeign. Interface declared with `@FeignClient`:

```java
@FeignClient(name = "ms-backoffice-service",
             url = "${feign.client.config.ms-backoffice-service.url}")
public interface IBackofficeFeignService {
    @GetMapping("/v1/config-parameters/ev-charging/settings")
    ResponseEvChargingFeatureSettings getEvChargingFeatureSettings(
        @RequestHeader("Operator-Id") int operatorCompanyId,
        @RequestHeader("Location-Id") int parkingLocationId);
}
```

**App setup:** `@EnableFeignClients` on the `@SpringBootApplication` class.

**`application.yml` config:**
```yaml
feign:
  client:
    config:
      ms-<name>-service:
        url: http://localhost:900X
      default:
        connectTimeout: 60000
        readTimeout: 62000
```

**Who talks to whom today (grepped 2026-07-01):**
- `ms-valet-service` is the primary outbound hub — has Feign clients to `ms-backoffice-service`, `ms-payment-service`, `ms-pms-service`, `ms-firebase-service`, `ms-notifications-service`.
- `ms-backoffice-service` has **NO** Feign clients as of the end of session. It's a receiver, not an initiator (we added one during survey work and rolled it back).
- Others: not audited yet.

**How to add Feign to ms-backoffice-service (if needed):**
1. Add dep to `pom.xml`: `<artifactId>spring-cloud-starter-openfeign</artifactId>` under `spring.cloud` group. Version comes from `spring-cloud-dependencies` BOM already in `<dependencyManagement>`.
2. Add `@EnableFeignClients` to `BackofficeServiceApplication`.
3. Create interface under `<feature>/feign/`.
4. Add `feign.client.config.<target-service>.url` to `application.yml` (and dev/prod profiles).
5. **Response DTOs need to mirror the target's JSON shape** — Jackson deserializes into whatever class the interface declares. Reuse existing wrappers (`Response`) and add feature-specific data fields.

## 2. RestTemplate — for external APIs

`ms-backoffice-service` has `RestTemplateConfig.java`. Used for:
- `OcraClient` (parking aggregator API, external)
- `NhtsaVehicleCatalogService` (NHTSA vehicle catalog, external)

**Not used for internal MS-to-MS.** If you're writing a new internal call, use Feign.

## 3. RabbitMQ / AMQP — async messaging

Both `ms-backoffice-service` and `ms-valet-service` have `spring-boot-starter-amqp`. In backoffice:
- `crm/config/RabbitConfig.java` — exchanges/queues declaration
- `crm/mq/GatewayTracerListener.java` — consumes gateway tracing
- `crm/toolskit/listeners/SystemEventsListener.java` — system events

**Use for:** audits, notifications, tracing, fire-and-forget events that don't need a response.

**Not for:** RPC-style calls (use Feign).

## 4. `ms-gateway-service` — edge router, not internal bus

Uses `spring-cloud-starter-gateway-server-webmvc`. Fronts the whole ecosystem — external clients (SPAs) hit `https://.../backoffice/*` etc. and it routes to the right MS. **Not** used as an internal service registry or discovery mechanism.

## Servlet context paths

Each MS has its own base path in the gateway config, but individual services do NOT declare `server.servlet.context-path`:
- `ms-backoffice-service` runs on port 9004, endpoints exposed at root (e.g. `/v1/surveys/locations-status`). The gateway prepends `/backoffice/` when routing external traffic.
- `ms-valet-service` runs on port 9000, endpoints at root (e.g. `/survey/questionnaire`).
- `ms-firebase-service` runs on 9010, `ms-payment-service` on 9001, `ms-pms-service` on 9005, `ms-notifications-service` on 9006.

**Feign clients target the internal port directly** (bypassing the gateway) via `feign.client.config.<name>.url = http://localhost:900X` (or the k8s service DNS name in dev/prod).

## How to apply

- If a feature needs data from another MS synchronously → **Feign** (add client to the caller, add `@EnableFeignClients` if not present, register URL in yml).
- If it's fire-and-forget or event-driven → **RabbitMQ** (add listener/publisher, declare exchange in RabbitConfig).
- If it's an external third-party API → **RestTemplate** (mirror `RestTemplateConfig`).
- If it's client-facing → controller under the gateway's routed path.
