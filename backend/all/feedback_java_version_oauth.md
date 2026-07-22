---
name: ms-oauth-service usa Java 21 (no 17)
description: ms-oauth-service es el único microservicio del backend en Java 21; el resto sigue en Java 17
type: feedback
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
**ms-oauth-service** se compila y corre con **Java 21**, mientras que los otros 7 microservicios usan Java 17.

**Why:** Decisión deliberada al extraerlo del monolito (Junio 2026), probablemente para aprovechar virtual threads, mejoras en Spring Security 6.x con records sealed, o features de Java 21. No alinearlo con el resto sin antes confirmar con el equipo.

**How to apply:**
- Al modificar el `pom.xml` de ms-oauth-service, mantener `<java.version>21</java.version>`. Al modificar cualquier otro servicio, mantener `<java.version>17</java.version>`.
- Si se propone alinear versiones, primero confirmar con el equipo qué features de Java 21 dependen.
- El Dockerfile de ms-oauth debe usar Corretto 21, no 17.
- Librerías como Lombok deben tener una versión compatible con Java 21 en ese servicio.
