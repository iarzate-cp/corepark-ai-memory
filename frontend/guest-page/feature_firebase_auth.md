---
name: Feature: Firebase Authentication for Guest Page
description: Arquitectura final, decisiones, e implementación de Firebase auth en el guest page — backend en ms-oauth-service, frontend en frontend-guest-page
type: project
originSessionId: eac5d57f-04cc-45e6-8c73-2c388f316d7a
---
## Objetivo

Agregar autenticación Firebase al guest page para habilitar escucha en tiempo real vía Firebase Realtime Database (estado del ticket/carro).

**Why:** La guest page necesita recibir actualizaciones en tiempo real del estado del ticket. Firebase Realtime Database ya es la infraestructura usada en otros apps de Corepark (frontend-validation, frontend-valet-web).

---

## Arquitectura final (confirmada 2026-06-19, refinada 2026-06-20)

```
Guest Page startup (paralelo)
    │
    ├── GET /partner/guest/ticket-info/{uuid}/{ticket}     (ms-valet-service)
    │
    └── GET /security/credentials/firebase                 (ms-oauth-service)
            │
            └── CredentialController.firebaseCredentials()
                    ├── utils.encrypt(env["google.firebase.user"], false)  → googleSecret
                    └── utils.encrypt(env["google.firebase.psw"],  false)  → googleSecret
                    └── retorna ResultBean estándar:
                        {
                          "code": "OK",
                          "message": "Successful operation",
                          "credentials": { "user": "...", "password": "..." }
                        }
            │
            └── FirebaseService.signIn({ user, password })
                    ├── decrypter(user,     environment.secretKey) → email
                    ├── decrypter(password, environment.secretKey) → password plano
                    └── signInWithEmailAndPassword(auth, email, password)
                            └── Firebase Realtime Database listener activo
```

### URL del endpoint en frontend
`environment.endpoint` ya apunta al mismo host que ms-oauth-service — no hay que agregar una URL nueva.
- dev:  `https://dev-web-71f618df0c78.corepark.com/security/credentials/firebase`
- prod: `https://api-web.corepark.com/security/credentials/firebase`

**Nota sobre el path (confirmado por el usuario 2026-06-20):**
- El gateway enruta `/security/*` → `ms-oauth-service` (stripping `/security` antes de pasar al servicio).
- `CredentialController` declara `@RequestMapping("/credentials")` → path interno `/credentials/firebase` → path público `/security/credentials/firebase`.
- `CommerceOtpController` declara `@RequestMapping("/oauth/commerce/otp")` → path público `/security/oauth/commerce/otp`.
- OAuth2 nativo de Spring → path público `/security/oauth/token`.
- En el frontend siempre llamar con el prefijo `/security/` para los endpoints de ms-oauth-service.

---

## Decisiones descartadas (historial)

| Opción | Razón de rechazo |
|--------|-----------------|
| Credenciales en `get-cfg` | Descartado por el equipo |
| Credenciales en `ticket-info` | Innecesario, se prefiere endpoint dedicado |
| `credentials-service` dedicado | Eliminado antes de implementarse |
| Lógica de encriptación en ms-valet-service | Duplicaría credenciales y Utils que ya viven en ms-oauth-service |
| Endpoint en ms-valet-service | ms-oauth-service es la única fuente de verdad de las credenciales |
| Path `/credentials/firebase-credentials` | Redundante — el GET ya define que son credenciales |

**Razón final de ms-oauth-service:** evita duplicar credenciales en yml files de múltiples servicios. Tradeoff aceptado: auth server con un endpoint público, ajustable en el futuro.

---

## Encriptación

**Algoritmo:** AES/ECB/PKCS5PADDING (Java) = AES/ECB/PKCS7 (CryptoJS) — equivalentes.

| Flujo | Clave | `isMobile` | Padding |
|-------|-------|------------|---------|
| Valet app | `.5m@rt.GuY.2020%` (`awsSecret`) | `true` | PKCS5/PKCS7 |
| Guest Page / Partner | `.F1r3B4$3-2o21#-` (`googleSecret`) | `false` | ISO10126 |

**Para la GP:** backend encripta con `isMobile=false` → `googleSecret`. Frontend desencripta con `environment.secretKey = '.F1r3B4$3-2o21#-'` que ya está en el guest page.

---

## Backend — implementación (ms-oauth-service)

**Branch:** `feature/firebase-guest-credentials`
**Repo:** `~/Dev/Back-End/ms-oauth-service`

### Archivos creados/modificados

#### 1. Bean: `FirebaseCredentials.java`
```
src/main/java/com/corepark/credentials/beans/response/FirebaseCredentials.java
```
```java
@Getter @Setter
public class FirebaseCredentials implements Serializable {
    private String user;
    private String password;  // (renamed from "psw" — 2026-06-20)
}
```

#### 2. Wrapper response: `FirebaseCredentialsResponseBean.java` (nuevo 2026-06-20)
Extiende `ResultBean` (que aporta `code`, `message`, `errors`) y agrega `credentials`.
```java
@Getter @Setter
public class FirebaseCredentialsResponseBean extends ResultBean {
    private FirebaseCredentials credentials;
}
```
**Patrón:** sigue a `PincodeRecoveryResponseBean`. Cumple con la convención de Corepark `{ code, message, ...payload }`.

#### 3. Endpoint en `CredentialController.java`
```java
@GetMapping("firebase")
public ResponseEntity<FirebaseCredentialsResponseBean> firebaseCredentials() {
    FirebaseCredentialsResponseBean response = new FirebaseCredentialsResponseBean();
    try {
        FirebaseCredentials credentials = new FirebaseCredentials();
        credentials.setUser(utils.encrypt(env.getProperty("google.firebase.user"), false));
        credentials.setPassword(utils.encrypt(env.getProperty("google.firebase.psw"), false));
        response.setCredentials(credentials);
        return new ResponseEntity<>(response, HttpStatus.OK);
    } catch (Exception e) {
        log.error("Failed to build Firebase credentials: {}", e.getMessage(), e);
        response.setCode(MessageCode.ERR_INTERNAL_SERVER.getCode());
        response.setMessage(MessageCode.ERR_INTERNAL_SERVER.getMessage());
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

### Commits en `feature/firebase-guest-credentials`
1. `feat: add firebase-credentials endpoint to CredentialController`
2. `fix: rename firebase endpoint path to remove redundant credentials segment`
3. `fix: add error handling to firebaseCredentials endpoint`

Ambas ramas tienen los tres commits: `feature/firebase-guest-credentials` y `feature/staging`.

### Credenciales en application.yml (confirmadas presentes)
```yaml
google:
  secret: .F1r3B4$3-2o21#-
  firebase:
    user: desarrollo@corepark.com
    psw: Qd2Ff8gJjD0@
```

---

## Frontend — implementación (frontend-guest-page)

**Branch:** `feature/firebase-auth`
**Repo:** `~/Dev/frontend-guest-page`

### Archivos creados/modificados

| Archivo | Estado | Cambio |
|---------|--------|--------|
| `core/models/enums/http-routes.ts` | ✅ | `FirebaseCredentials = '/security/credentials/firebase'` |
| `core/services/oauth-service.ts` | ✅ | `getFirebaseCredentials()` → mapea `.credentials` del wrapper `{ code, message, credentials: { user, password } }` |
| `core/models/definitions/env.ts` | ✅ | Agregada `FirebaseConfig` interface + props `secretKey` y `firebaseConfig` en `Environment` |
| `src/environments/environment.ts` + `.dev.ts` + `.prod.ts` | ✅ | `secretKey: '.F1r3B4$3-2o21#-'` + `firebaseConfig` (dev → `corepark-services-dev`, prod → `corepark-services`) |
| `core/utils/decrypter.ts` | ✅ | Copia idéntica de valet-web (AES/ECB/PKCS7/Base64 con `env.secretKey`) — nombre `decrypter` no `decryptor` para alinear con valet-web |
| `core/services/firebase-service.ts` | ✅ | `signIn({ user, password })` desencripta + `signInWithEmailAndPassword`; `listenTicket({ operatorCompanyId, parkingLocationId, ticketId })` con `onValue` envuelto en Observable + `ngZone.run()` |
| `core/states/firebase-state.ts` | ✅ | `ticket` signal — `toSignal(toObservable(#listenParams).pipe(switchMap(...)))` reactivo a `ticketInfoState.ticketInfo` |
| `core/initializers/firebase-credentials.ts` | ✅ | **Retorna** el observable (bloquea bootstrap hasta signIn) — `getFirebaseCredentials → switchMap(signIn) → catchError` |
| `app.config.ts` | ✅ | `initializeApp(env.firebaseConfig)` + `provideFirebaseApp`, `provideAuth`, `provideDatabase` |
| `app.component.ts` | ✅ | Inyecta `FirebaseState` + effect que loguea `ticket` updates |
| `package.json` | ✅ | `crypto-js@^4.2.0` + `@angular/fire@^19.0.0` (sin `@types/crypto-js` — funciona porque `strict: false`) |

### Firebase config — dev (idéntico para `environment.ts` y `environment.dev.ts`)
```ts
firebaseConfig: {
  apiKey: 'AIzaSyDQpplpxP7_WgPyG-CA12z_awusn2NBOIc',
  authDomain: 'corepark-services.firebaseapp.com',
  databaseURL: 'https://corepark-services-dev.firebaseio.com',
  projectId: 'corepark-services-dev',
  storageBucket: 'corepark-services-dev.appspot.com',
  messagingSenderId: '582077607268',
  appId: '1:582077607268:web:f4aea7aecf229e6fb81e71',
  measurementId: 'G-XPSSJMYMY4',
}
```
Prod cambia `databaseURL`, `projectId`, `storageBucket` a `corepark-services` (sin `-dev`).

### Path de Firebase RT DB (escuchado en GP)
```
parkingServices/operatorCompany/{operatorCompanyId}/parkingLot/{parkingLocationId}/ticket/{ticketId}
```
**Nota:** valet-web escucha el nodo padre `/ticket` (lista completa); GP escucha el ticket específico por `ticketId`. `operatorCompanyId` y `parkingLocationId` vienen de `ticketInfoState.ticketInfo().company.operatorId` y `.location.id`; `ticketId` de `ticketInfo.ticket.ticket`.

### Decisión: bloquear bootstrap en signIn
El initializer **retorna** el observable de `signIn` para que Angular espere antes del bootstrap. Sin esto, el listener de RT DB intentaría leer antes de que auth esté listo y Firebase rechazaría la lectura. Trade-off aceptado: un poco más de latencia inicial a cambio de no manejar el race manualmente.

### Archivos de referencia en frontend-valet-web
| Archivo | Descripción |
|---------|-------------|
| `src/app/core/utils/decrypter.ts` | AES/ECB/PKCS7/Base64 con secretKey — copiado tal cual |
| `src/app/core/services/firebase-service.ts` | Patrón `signIn()` |
| `src/app/core/states/firebase-state.ts` | Patrón `toSignal(toObservable(...).pipe(switchMap(...)))` |
| `src/app/core/services/tickets-service.ts` | Patrón `onValue()` en Observable manual con `ngZone.run()` |
| `src/app/app.component.ts` | signIn se trigerea desde app.component reactivo (GP lo hace en initializer en su lugar) |

---

## Estado actual (2026-06-20)

### Backend (ms-oauth-service)
- ✅ `FirebaseCredentials.java` con campos `user` / `password`
- ✅ `FirebaseCredentialsResponseBean extends ResultBean` (patrón Corepark `{ code, message, credentials }`)
- ✅ Endpoint `GET /credentials/firebase` (público: `/security/credentials/firebase`) en `CredentialController`
- ✅ Error handling con `MessageCode.ERR_INTERNAL_SERVER`
- ✅ Endpoint verificado desde el browser

### Frontend (frontend-guest-page) — branch `feature/firebase-auth`
- ✅ `crypto-js` y `@angular/fire` instalados
- ✅ `secretKey` + `firebaseConfig` en los 3 env files (con `FirebaseConfig` interface tipada)
- ✅ `decrypter.ts` — desencriptación verificada (devuelve `desarrollo@corepark.com` y password en claro)
- ✅ `FirebaseService.signIn()` — sign-in verificado (`[firebase] signed in` en consola)
- ✅ `provideFirebaseApp` + `provideAuth` + `provideDatabase` en `app.config.ts`
- ✅ Initializer bloquea bootstrap hasta que signIn resuelva
- ✅ `FirebaseService.listenTicket()` — `onValue` en Observable con `ngZone.run()`
- ✅ `FirebaseState.ticket` signal reactivo a `ticketInfoState.ticketInfo`
- ✅ `app.component.ts` loguea ticket updates vía effect

### Pendientes (próximos pasos)
- ⏳ Definir qué hacer con los datos de RT DB (qué campos consume la UI, refresh del ticket info, triggers de notificaciones, etc.)
- ⏳ Tipar el snapshot del ticket (`unknown` → interface real cuando se decida qué consumir)
- ⏳ Reemplazar `console.log` del initializer y de `app.component` por flujo productivo
- ⏳ Decidir si extraer un `tickets-service.ts` separado o mantener el listener en `firebase-service.ts`
