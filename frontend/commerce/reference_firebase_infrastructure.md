---
name: Corepark Firebase and microservice routing
description: Firebase RTDB instances per environment, ms-firebase-service routing conventions, key data paths in the Realtime DB, and console access for verification
type: reference
originSessionId: a2019106-3f08-4f27-8756-0bb019dfebd1
---
# Firebase infrastructure and cross-service routing

## Firebase Realtime DB instances

- **Dev:** `https://corepark-services-dev.firebaseio.com`
- **Prod:** `https://corepark-services.firebaseio.com/`

Credentials JSON (both envs): `corepark-services-firebase-adminsdk.json`. Lives only in `~/Dev/Back-End/ms-firebase-service/src/main/resources/` (and in the deployed image). Not committed to any other repo — everyone else routes through ms-firebase-service via REST.

## ms-firebase-service routing between services

Two client-side conventions coexist in the codebase — not unified yet:

- **ms-valet-service (Feign):** `feign.client.config.ms-firebase-service.url`
  - Dev: `http://ms-firebase-service.${NAMESPACE}` (K8s namespace env var)
  - Prod: `http://ms-firebase-service.internal-corepark.com`
- **ms-backoffice-service (RestTemplate):** `service.firebase.url`
  - Local default: `http://localhost:9010`
  - Dev/prod: **not defined in the yml as of 2026-07-06** — inherit the localhost default which is broken in the cluster. Fixing this is the current release blocker.

## Ports

- ms-firebase-service local: 9010
- ms-firebase-service dev/prod: 80

## Key data paths in Firebase Realtime DB

- **Active tickets (carlist):** `/parkingServices/operatorCompany/{opId}/parkingLot/{locId}/ticket/`
- **Historical tickets (post-checkout):** `/parkingServices/recentTickets/`
- **Postings:** `/parkingServices/postings/{postingUuid}`

Tickets move from the carlist to `recentTickets` on check-out — the carlist entry is deleted, not flagged with a checkOut timestamp.

## Firebase console access

For validation during dev work: log into Firebase console → project `corepark-services-dev` → Realtime Database → navigate the tree. Deletes done from ms-firebase-service DELETE endpoints are visible here immediately.
