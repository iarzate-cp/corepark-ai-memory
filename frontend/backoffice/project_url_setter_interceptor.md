---
name: URL setter interceptor + endpoints unificado
description: Refactor que centraliza URLs vía interceptor HTTP + objeto endpoints único; rama lista pero NO en producción todavía
type: project
originSessionId: dcd7e602-69f1-49b6-9846-55f5d103065d
---
Rama `feature/interceptor/url-setter` lista pero **NO mergeada a main todavía**. El usuario va a hacer otras tareas primero y volver a cerrar esta después.

**Qué cambió en la rama:**
- Nuevo `core/interceptors/url-setter-interceptor.ts` que antepone `environment.endpoint` cuando `req.url` empieza con `/`. Registrado primero en `withInterceptors([...])` en `app.config.ts`.
- Nuevo `core/http/endpoints.ts` único con TODAS las URLs agrupadas por dominio. Estáticas → `endpoints.foo.bar` (string), dinámicas → `endpoints.foo.byId(id)` (función).
- ~30 servicios migrados: ya no construyen URL con `pathSetter(...)` ni `${environment.endpoint}...` — todos usan referencias del nuevo objeto.

**Eliminado:**
- `core/http/path-setter.ts`
- `core/http/endpoints/` (directorio con sub-archivos por dominio)
- `core/utilities/router.utilities.ts` (`apiUrl`)
- `core/interfaces/routing/api-routes.interfaces.ts` (enums duplicados)

**Estado:**
- `tsc --noEmit` pasa sin errores
- No probado en navegador todavía
- El usuario maneja git manualmente — pendiente de él commit, push y PR contra main

**Why:** Había tres convenciones compitiendo (`pathSetter` + `apiUrl` + inline strings + enums duplicados con objetos `endpoints` dispersos). El usuario quería una sola fuente de verdad y eliminar boilerplate de construir URLs en cada servicio.

**How to apply:** Si surge una tarea en esta rama, los nuevos servicios deben usar `endpoints.X.Y` para URLs estáticas y `endpoints.X.byId(...)` para dinámicas — nunca inline strings ni reintroducir `pathSetter`. Si el usuario menciona "el interceptor" o "los endpoints unificados", se refiere a esto. Antes de recomendarle merge, verificar con él que ya probó en navegador.
