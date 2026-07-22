---
name: Corepark backend microservices architecture
description: The 9 Java Spring Boot microservices, the API gateway pattern (all traffic entering :8080 and routed by path prefix), local/dev/prod URI conventions, StripPrefix rules, and JWT auth model. Foundational reference — every backend interaction goes through this
type: reference
originSessionId: a2019106-3f08-4f27-8756-0bb019dfebd1
---
# Corepark backend — microservices architecture

## The gateway pattern (critical)

**All external traffic enters through `ms-gateway-service` on port `:8080`.** Frontends, mobile apps, and external clients never talk directly to individual microservices. The gateway routes requests by path prefix to the appropriate downstream micro, then forwards the response back. Locally, dev, and prod all expose port 8080 to clients — only the downstream URIs change per environment.

Stack: Spring Cloud Gateway (WebMvc, sync — not the reactive WebFlux variant). Java 21, Spring Boot 3.5.15, Spring Cloud 2025.0.2. Configured entirely in `spring.cloud.gateway.server.webmvc.routes` in `application*.yml`.

Service discovery is **hard-coded YAML per environment**:
- **Local:** `http://127.0.0.1:<port>` for each downstream micro.
- **Dev:** `http://ms-<service>-service.${NAMESPACE}` (K8s DNS with the `NAMESPACE` env var).
- **Prod:** `http://ms-<service>-service.internal-corepark.com` (private internal DNS).

Spring's `discovery.enabled: false` — there is no Eureka/Consul.

## Routing table (source of truth)

The **`StripPrefix` filter matters** — it changes what URL path the downstream service sees. Anything with `StripPrefix=1` means the first path segment (the prefix) is removed before forwarding. If the downstream controller expects the prefix, you get a 404 with `No static resource ...` from Spring's `NoResourceFoundException`.

| Gateway prefix | StripPrefix | Downstream micro | Local port | Meaning for downstream controllers |
|---|---|---|---|---|
| `/backoffice/**` | `1` | ms-backoffice-service | 9004 | Controllers must NOT include `/backoffice` in their `@RequestMapping` — the gateway strips it. Use `/v1/...`, `/firebase-cleanup/...`, etc. |
| `/crm/**` | none | ms-backoffice-service | 9004 | Controllers keep the `/crm` prefix — e.g., `@RequestMapping("/crm/settings/valet-app")`. |
| `/security/**` | `1` | ms-oauth-service | 9007 | e.g., `POST /security/oauth/token` at gateway → `POST /oauth/token` at oauth. |
| `/credentials/**` | none | ms-oauth-service | 9007 | Controllers keep `/credentials`. |
| `/valet/**` | `1` | ms-valet-service | 9000 | Strip. |
| `/partner/**` | none | ms-valet-service | 9000 | Kept. Note: partner + valet both route to the same micro. |
| `/payment/**` | `1` | ms-payment-service | 9001 | (Payment repo not in `~/Dev/Back-End/`, see prod deploy.) |
| `/reports/**` | `1` | microservice-reports | 9002 | Strip. |
| `/notifications/**` | `1` | ms-notifications-service | 9006 | Strip. |
| `/pms/**` | `1` | ms-pms-service | 9005 | Strip. |
| `/firebase/*` | *no explicit route* | ms-firebase-service | 9010 | Not exposed via gateway; only inter-service calls. |

**Golden rule:** before adding a new backoffice endpoint, check whether it will be reached under `/backoffice/**` (StripPrefix=1 — controller path must NOT start with `/backoffice`) or under `/crm/**` (no strip — controller path INCLUDES `/crm`).

## Microservice roster

Everything under `~/Dev/Back-End/`. All are Spring Boot, Java 21, PostgreSQL (local `127.0.0.1:5432/coreparkdev`, dev/prod on AWS RDS). All publish/consume RabbitMQ events on exchange `direct.amq.corepark`.

| Repo | Local port | Domain / responsibility |
|---|---|---|
| `ms-gateway-service` | 8080 | API gateway. See above. |
| `ms-oauth-service` | 9007 | Authentication. Emits JWTs signed with RS256 (RSA key pair hardcoded in `JwtConfig.java`). Handles token issuance, OTP flow for commerce, password recovery. Also owns the `/credentials/**` endpoints for AWS creds, Firebase creds, valet pincode login. |
| `ms-backoffice-service` | 9004 | Backoffice admin domain + CRM. Everything under `/backoffice/**` and `/crm/**`. Owns settings, valet-app config, aggregators, chargers, screen config, damage config, survey, UUID table, and (this feature) `firebasecleanup`. |
| `ms-valet-service` | 9000 | Valet operational domain — check-in, check-out, ticket lifecycle, partner-facing endpoints under `/partner/**`. Produces the Firebase carlist via RabbitMQ events (root cause of orphan tickets when the check-out event fails). |
| `ms-firebase-service` | 9010 | Only service allowed to speak the Firebase Admin SDK directly. Not exposed via gateway. Path in RTDB: `/parkingServices/operatorCompany/{opId}/parkingLot/{locId}/ticket/`. See `reference_firebase_infrastructure.md`. |
| `ms-pms-service` | 9005 | PMS integrations (Opera Cloud/On-Prem, StayNTouch, Guesty, Infor), arrivals file processing via email + Claude API for ML parsing, AWS SQS for hotel-response fan-out. |
| `ms-notifications-service` | 9006 | Notification hub — email (Gmail SMTP), SMS (Twilio), chat. Consumes RabbitMQ queues `email-service` / `sms-service` from other micros. |
| `ms-partner-service` | 9009 | Partner-facing ticket validation, compensation, hotel ticket info. |
| `microservice-reports` | 9002 | Reporting engine — overnight occupancy, revenue & labor, employee performance, SMS usage, inventory. CSV/XLS export endpoints. |

## Auth flow across the stack

**Token issuance:** Frontend commerce calls the oauth service through the gateway (`POST /security/oauth/commerce/otp/request` → `send` → `verify`). On `/verify` the oauth service returns a JWT (RS256, ~1h expiry for the access token, 48h in dev/prod for commerce specifically).

**Token validation:** Done **at the gateway**, not in each micro. The gateway has the RSA public key hardcoded in `JwtConfig.java` and uses `NimbusJwtDecoder` to verify the signature and expiration. If valid, the request is forwarded downstream with the `Authorization: Bearer ...` header intact. Downstream micros trust the gateway and don't re-verify.

**Public routes (no token needed):** allow-listed at the gateway per host (web vs mobile channel):
- Always public: `/health`, `/security/oauth/**`.
- Web channel additional: webhooks (Twilio, Windcave, Freedompay, Square, PMS webhooks), guest-page endpoints, `/credentials/backoffice-password-recovery`, `/backoffice/survey/questionnaire`, `/partner/guest/**`, `/security/credentials/firebase`, and a few more (see `ResourceServerConfig.webSecurityFilterChain`).

**Two host-based channels:** the gateway distinguishes "mobile" vs "web" by the `Host` header. Different security chains apply. Local mobile-host is `api-mobile.localhost`; local web-host is `localhost`. In dev/prod they map to `dev-mobile-*.corepark.com` / `dev-web-*.corepark.com` and their prod equivalents.

**Operator claim validation (mobile only):** mobile requests must have `Operator-Id` header (or `operatorCompanyId` in body) matching the JWT's `metadata.operatorCompanyId`, or the gateway returns 403 `OPERATION-NOT-ALLOWED`.

## CORS (only enforced at the gateway)

Configured in `CorsConfig.java` + `application*.yml` `cors.allowed-origins`. Local allows `https://localhost:4400`. Dev allows all local dev ports plus dev CloudFront distributions. Prod allows the production `corepark.com/mx/ca` subdomains and CDNs.

Headers whitelisted: `Content-Type`, `Authorization`, `Operator-Id`, `Location-Id`, `Partner-Id`, `ReCaptcha-Token`. `allowCredentials: true`.

## Extra gateway filters worth knowing

- **`LogFilter`** (~546 lines) — the workhorse. Adds `App-Trace-Id` header, logs request/response bodies (with 64KB cap and streaming for large payloads), publishes error events to RabbitMQ (`system-monitoring` queue) on 500/415/405 and connection failures (which surface as 503 to the caller), and traces valet check-in flows to `gateway-tracer` queue.
- **`ReCaptchaValidationFilter`** — web channel only, protects specific payment endpoints (`/payment/freedompay/hpc/init` / `submit`). Requires `ReCaptcha-Token` header, verifies with Google API, threshold 0.5.
- **Security headers** — `Content-Security-Policy: default-src 'self'`, `X-Frame-Options: DENY`, HSTS 1 year, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()`.

## Local dev — how to run the stack

For any feature that traverses the gateway, at minimum you need: PostgreSQL (`coreparkdev` DB on localhost:5432), the downstream micro(s), and the gateway. Optionally RabbitMQ if you want the async events / monitoring; the gateway doesn't hard-block on it being down.

Startup order matters: bring up the downstream micros first (they're what the gateway will forward to), then the gateway. If a downstream is not listening, the gateway returns 503 for that route (and pushes an alert to RabbitMQ if configured).

Datasources default to `localhost:5432/coreparkdev` in every micro's base `application.yml`. To point at dev RDS instead, run with `-Dspring.profiles.active=dev`. Note that some micros' `application-dev.yml` also change the port to 80 (the K8s container-internal port), so you may need `-Dserver.port=<port>` overrides if you want dev-profile config on a specific local port.

## Where docs used to be missing

Neither the gateway repo nor most of the micros carries a README or `docs/` folder. Architecture was tribal knowledge until this memory was written on 2026-07-06. If future changes to the gateway routing table happen, this memory needs a corresponding update.
