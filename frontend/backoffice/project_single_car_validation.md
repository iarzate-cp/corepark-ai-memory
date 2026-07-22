---
name: Validación — no borrar carro si es el único del guest profile
description: Restricción coordinada front+back que impide dejar un guest profile sin carros; diseño original, posiblemente obsoleta
type: project
originSessionId: 15f1cd0b-724b-4750-8329-9c59d168c9ad
---
Un guest profile no puede quedar sin carros: si solo tiene uno, el delete está bloqueado en ambas capas.

**Backend:** `CarProfilesServiceImpl.java` línea 152 — consulta `getCarCountByCarId(carProfileId)` y si `carsCount <= 1` retorna `ERR_BS_INVALID_GUEST_PROFILE_01` sin ejecutar el delete.

**Frontend:** Guard equivalente que evita llamar al endpoint cuando solo hay un carro — mejora UX evitando el round-trip.

**Why:** Decisión del desarrollador original (ya no está en el equipo). En la versión inicial, guest profile y primer carro se creaban juntos — un perfil sin carros era un estado imposible. No hay comentario explicativo en el código, solo el error code sugiere que era una regla de negocio formal.

**How to apply:** Con el nuevo endpoint `POST /backoffice/car-profiles` (add-car) el perfil y los carros ya son entidades separadas, por lo que esta restricción puede ser candidata a eliminarse. Si se decide quitarla: remover la validación `carsCount <= 1` en línea 152 del `CarProfilesServiceImpl` y el guard correspondiente en el front. De momento se deja intacta — no hay urgencia.
