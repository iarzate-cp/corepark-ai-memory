---
name: Docs ms-oauth-service
description: OAuth 2.0 Password Grant + JWT + recuperación de credenciales + AWS creds + OTP. Puerto 9007. Java 21
type: project
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
**Puerto:** 9007 · **Stack:** **Java 21** (¡distinto al resto!) + Spring Boot 3.5.15 · **App name:** `ms-oauth-service` · **max-http-request-header-size:** 512KB

**Responsabilidad:** Servidor de autenticación centralizado. Emisión y validación de JWT bajo OAuth 2.0 Password Grant. Gestión de credenciales (recuperación de password, pincode login, credenciales AWS para clientes mobile/web). OTP para comercio (Square).

**Clientes OAuth configurados (en `application.yml`):**
- `valet-client` (apps valet) — access token 3600s, refresh 4000s
- `partner-client` (apps de socios/hoteles) — access token 3600s
- (otros pueden agregarse en `config.security.oauth`)

**Estructura por dominio:**
- `com.corepark.oauth.controller.TokenController` — POST `/token` (consume `application/x-www-form-urlencoded`)
- `com.corepark.oauth.controller.OauthController` — GET `/health`
- `com.corepark.oauth.controller.CommerceOtpController` — POST `/request`, `/send`, `/verify` (flujo OTP)
- `com.corepark.credentials.controller.CredentialController` — POST `backoffice-password-recovery`, PUT `update-password`, POST `login-pincode`, POST `valet-pass-recovery`, POST `location-pass-recovery`, GET `mobile-aws-cred` / `web-aws-cred` / `firebase`
- `com.corepark.oauth.service.TokenEnhancerService` — añade metadata custom (LightweightValetMetadata) a tokens

**Dependencias clave:** spring-boot-starter-security, nimbus-jose-jwt, bcprov-jdk18on (Bouncy Castle), commons-fileupload, springdoc-openapi.

**Submódulo `credentials/beans/pms/oracle/`:** sugiere que también gestiona credenciales de integraciones PMS Oracle/Opera.

**Why:** Servicio NUEVO (Junio 2026), extraído para centralizar todo lo que es autenticación, recuperación de credenciales y gateways de AWS/Firebase para clientes finales. Java 21 deliberadamente — para aprovechar virtual threads / mejoras de Spring Security.

**How to apply:** Cualquier flujo de login, refresh token, recuperación de password o emisión de credenciales temporales AWS para clientes mobile/web pasa por aquí. Si algo necesita validar un JWT, debe contra este servicio. **Cuidado al cambiar dependencias:** la app usa Java 21, no 17.
