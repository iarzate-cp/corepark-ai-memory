# corepark-ai-memory

Consolidación local de todas las memorias que Claude Code ha ido acumulando por
proyecto de Corepark. Cada carpeta es un espejo del directorio `memory/` del
proyecto correspondiente bajo `~/.claude/projects/…/memory/`.

## Estructura

```
backend/
├── all/                              # memorias del root ~/Dev/Back-End (cross-microservicio)
├── ms-backoffice-service/            # microservicio backoffice
├── ddl-coupons/                      # release DDL 2026-07-15 (COUPONS)
└── ddl-coupon-request-backfill/      # release DDL 2026-07-22 (backfill CREATE)

frontend/
├── backoffice/
├── commerce/
├── guest-page/
├── resident-portal/
├── validation/
└── valet-web/

design-system/                 # librería de UI compartida
shared/                        # memorias cross-proyecto (root ~/Dev)
aws-tunnel/                    # sesiones en ~/Documents/AWS (tunnel + ops AWS)
```

Cada carpeta conserva su propio `MEMORY.md` como índice interno.

## Convenciones

- `project_*.md` — contexto de trabajo, iniciativas, decisiones.
- `reference_*.md` — punteros a sistemas/recursos externos.
- `feedback_*.md` — guía sobre cómo trabajar (do/don't).
- `user_*.md` — perfil del usuario, preferencias.
- `docs_*.md` — documentación técnica de un microservicio/módulo.

## Cómo sincronizar de vuelta

Estos archivos son la copia consolidada. La fuente de verdad para el
runtime de Claude Code sigue siendo `~/.claude/projects/<slug>/memory/`.
Si editas algo aquí, hay que copiarlo de vuelta al `memory/` del proyecto
correspondiente para que Claude lo lea en la siguiente sesión.

## Pendientes de seguridad

- [ ] **Rotar** el AWS Access Key ID que estaba en
      `backend/all/docs_ms_backoffice_service.md` (ya redactado). Estuvo en
      notas locales — asumir comprometido y revocar/rotar en IAM.
- [ ] **Rotar** el Twilio Account SID que estaba en
      `backend/all/docs_ms_notifications_service.md` (ya redactado).
- [ ] Rotar / restringir por dominio HTTP Referrer las 3 API keys de Google
      (Firebase apiKey y las dos de Google Maps DEV/PROD).
- [ ] Evaluar rotación de `awsSecret` y `googleSecret` (llaves de cifrado
      client-side referenciadas en `frontend/guest-page/feature_firebase_auth.md`).
- [ ] Los endpoints de RDS DEV y MQ DEV están citados en `backend/all/`;
      no son secretos pero exponen topología — revisar si conviene enmascarar.
