---
name: Los tests de contexto necesitan el túnel a la BD abierto
description: ValetServiceApplicationTests y cualquier @SpringBootTest que levante el contexto fallan con "Failed to obtain JDBC Connection" si el túnel a localhost:5432 no está arriba — no es una regresión
metadata:
  type: reference
---
`application.yml` apunta a `jdbc:postgresql://localhost:5432/coreparkdev`, y `InitialCatalogLoading` consulta la BD **al levantar el contexto**. Entonces cualquier `@SpringBootTest` que arranque la app entera falla así cuando el túnel está cerrado:

```
UnsatisfiedDependencyException: Error creating bean with name 'valetController'
  → 'getRequestsStatus' defined in InitialCatalogLoading
  → CannotGetJdbcConnectionException: Failed to obtain JDBC Connection
```

Confirmado en `ms-valet-service` (`ValetServiceApplicationTests`) el 2026-09-02, y aplica a cualquier microservicio con el mismo arranque.

**No es una regresión del cambio que estés haciendo.** Antes de dudar del propio código:

1. `lsof -Pi :5432 -sTCP:LISTEN -t` — si no devuelve nada, el túnel está cerrado y ese es el motivo.
2. Para atribuirlo con certeza: `git stash push -u`, correr el mismo test sobre el HEAD limpio, `git stash pop`. Si falla igual, no es tuyo.

El túnel se abre con `~/Documents/AWS/tunnel.sh dev` (deja `localhost:5432` RW y `6432` RO).

## Ojo con la base de la rama

Los tests que fallan también dependen de **de dónde salió la rama**. El 2026-09-02, una rama nacida de `main` traía además `TicketsServiceCancelRequestTest.cancelRequest_NonScheduledRequest_SendsCorrectSmsTemplate` roto, mientras que en `feature/staging` —que iba más adelante— la suite pasaba entera con 143 tests. Comparar contra la rama destino antes de asumir que rompiste algo.
