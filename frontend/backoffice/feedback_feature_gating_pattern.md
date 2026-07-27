---
name: Feature gating por operador — guard + nav filter + prod-only allowlist
description: Patrón de dos capas para restringir features a operadores específicos; predicate único compartido, bypass en no-prod
type: feedback
originSessionId: 5506e125-b5d9-4d50-a707-64609419d5d0
---
Cuando una feature debe restringirse a un subconjunto de operadores en producción, usa el patrón de dos capas que ya está en `activity-by-rate-class`:

1. **Predicate único** en `core/utils/<feature>-access.ts`:
   ```ts
   export const canAccess<Feature> = (operatorCompanyId: number | null): boolean => {
     if (!environment.production) return true          // no-prod: todos ven la feature
     if (operatorCompanyId === null) return false      // sin operator resuelto: bloquea
     return ALLOWED_OPERATORS.includes(operatorCompanyId)
   }
   ```
2. **Route guard** en `core/guards/<feature>/<feature>.guard.ts` que consume el predicate vía `toObservable(operatorDataState.operatorCompanyId)` y redirige al fallback (`router.createUrlTree([...])`).
3. **Nav filter** en `main-layout-nav.routes.ts::getAppRoutes(operatorCompanyId)` que remueve la entrada del sidebar/urlTree usando el mismo predicate.

**Why:** Solo esconder el link del sidebar no basta — un deep link a la URL pasa. Solo el guard tampoco basta — la entrada visible confunde a operadores no autorizados. Ambas capas juntas dan defense-in-depth con un solo predicate como fuente de verdad. El bypass en no-prod evita que un dev en staging con un operador random no pueda probar la feature.

**How to apply:**
- Reutilizar `ProductionOperators` (enum en `core/enums/operators.ts`, CorePark=1, SecureParking=2, etc.) para la lista de acceso.
- Cablear el guard en `app.routes.ts` con `canActivate: [<feature>Guard]`.
- El nav filter va en `getAppRoutes` (ya existe la función); agrégale un branch para la nueva feature en vez de duplicar el mapeo.
- No mover el predicate al guard directamente — al ser una función pura, se testea aparte y se reusa en el nav sin importar Angular.
