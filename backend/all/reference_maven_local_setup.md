---
name: Maven local — el wrapper NO sirve, y ojo con el modo offline
description: ./mvnw resuelve Maven 3.6.0 y truena con maven-compiler-plugin 3.14.1; usar el 3.9.9 del cache de wrapper. Y no correr con -o cuando la rama trajo dependencias nuevas.
type: reference
---

## Trampa 1 — `./mvnw` está roto en local

En `microservice-reports` (y probablemente en el resto de los `ms-*`):

- `mvnw` **no tiene bit de ejecución** → `permission denied`. Hay que invocarlo como `sh ./mvnw`.
- `maven-wrapper.properties` apunta a **Maven 3.6.0**, y el `pom.xml` usa `maven-compiler-plugin:3.14.1`, que **exige Maven ≥ 3.6.3**. Resultado: `PluginIncompatibleException`, nunca compila.
- No hay `mvn` en el PATH (no está instalado por Homebrew).

**La salida:** el cache del wrapper ya tiene versiones nuevas descargadas de otros proyectos. Usar el 3.9.9 directo:

```sh
~/.m2/wrapper/dists/apache-maven-3.9.9-bin/*/apache-maven-3.9.9/bin/mvn -DskipTests compile
```

(Hay 3.6.0, 3.6.2, 3.6.3, 3.8.6 y 3.9.9 en `~/.m2/wrapper/dists/`.)

Java local: Corretto 21.0.11, ya en el PATH.

## Trampa 2 — `-o` (offline) truena cuando la rama trajo dependencias nuevas

Correr offline es más rápido y suele funcionar, pero si acabas de mergear una rama que agregó dependencias, éstas no están en `~/.m2` y el build muere con `Cannot access central ... in offline mode`.

Caso real 2026-08-13: mergear el External Data API v1 a `feature/staging` trajo `springdoc-openapi-starter-webmvc-api:2.9.0` y `spring-cloud-aws-starter-secrets-manager:3.4.2`. La suite falló con `-o` y pasó sin la bandera.

**Regla:** después de mergear trabajo ajeno, correr **sin** `-o` la primera vez.

## Referencia de tiempos

Suite completa de `microservice-reports`: 224 tests, ~3 min con descarga de deps, mucho menos en caliente.
