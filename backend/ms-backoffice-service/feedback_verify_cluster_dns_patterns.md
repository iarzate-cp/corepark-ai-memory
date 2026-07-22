---
name: Verify cluster DNS patterns before proposing env-specific URLs
description: When adding a new service URL to application-{env}.yml, copy the DNS pattern from an existing analogous entry — do not invent placeholders like ${NAMESPACE} or assume conventions
type: feedback
originSessionId: e093e519-e068-4f95-9135-b45cf1bf103f
---
When overriding a downstream microservice URL in `application-dev.yml` or `application-prod.yml`, always mirror the exact DNS shape of an already-deployed sibling entry in the same yml.

**Why:** On 2026-07-06 I proposed `http://ms-firebase-service.${NAMESPACE}` for the dev override, assuming `NAMESPACE` was a K8s-injected env var (a pattern common in some Helm setups but not in this cluster). The placeholder never resolved and the deploy failed. The team's actual convention is a literal DNS: `ms-firebase-service.dev.internal-corepark.com` for dev, `ms-firebase-service.internal-corepark.com` for prod — same shape as everything else in the yml (Ocra, Spothero, Parkcheep, RDS all use literal DNS with the env baked in the subdomain, not a `${VAR}` placeholder).

**How to apply:**
- Before proposing any env-specific URL, `grep` the same yml (`application-dev.yml` / `application-prod.yml`) for other `url:` entries and match their literal shape.
- Never introduce a `${VAR}` unless you can point to an existing entry in the same yml that uses it AND to the env-var declaration in the chart/values or deployment manifest.
- For internal microservice URLs specifically: the cluster convention is `http://ms-<name>-service[.<env>].internal-corepark.com`. Dev has the `.dev.` segment; prod does not.
- When unsure, ask the team explicitly *"what's the DNS for ms-X-service in dev?"* rather than inferring from generic Kubernetes patterns — different clusters expose DNS differently and my priors from K8s tutorials do not automatically match Corepark's setup.
