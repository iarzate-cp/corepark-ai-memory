---
name: Bug — DELETE car-profile 500 en autos sin fotos (RESUELTO)
description: Fix aplicado en CarProfilesServiceImpl línea 166; pendiente commit+PR del usuario
type: project
originSessionId: 15f1cd0b-724b-4750-8329-9c59d168c9ad
---
`DELETE /backoffice/car-profiles/:id` devolvía 500 cuando el auto no tenía fotos.

**Why:** `deletePhotoReels([])` ejecutaba `WHERE id IN (:photoIds)` con lista vacía → Spring JDBC lanzaba `InvalidDataAccessApiUsageException` antes de llegar al delete del auto.

**Fix aplicado** en `ms-backoffice-service/src/main/java/com/corepark/backoffice/guestprofile/service/impl/CarProfilesServiceImpl.java` línea 166:

```java
// Antes (bug):
final boolean isPhotoReelDeleted = guestProfilesDao.deletePhotoReels(photosIds);

// Después (fix):
final boolean isPhotoReelDeleted = photosIds.isEmpty()
        || guestProfilesDao.deletePhotoReels(photosIds);
```

**How to apply:** Fix ya en rama `fix/car-profile-delete-empty-photo-reel`. Afectaba todos los autos sin fotos, incluyendo los creados con el nuevo `POST /backoffice/car-profiles`. Commit y PR pendientes de que el usuario los haga manualmente.
