---
name: sesion-del-2026-08-27-arco-y-orden-de-lo-hecho
description: índice cronológico de una sesión larga que fue de quitar Tailwind del DS a migrar el rail del backoffice; los detalles viven en las memorias enlazadas
metadata: 
  node_type: memory
  type: project
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-27T23:48:36.774Z
---

Sesión larga en `~/Dev/design-system` que acabó tocando **cuatro repos**. Los detalles están por tema en otras memorias; esto es el orden y la causalidad, que no se ve leyéndolas por separado.

## El arco

1. **Quitar Tailwind del design system.** Punto de partida: "no lo estamos usando y solo aporta espacio". Resultó que el CSS de Tailwind vivía **solo en el demo**, así que 4 componentes de la librería enviaban clases sin CSS a los consumidores — `cp-radio-*` estaba roto en commerce. Se migraron a BEM SCSS. → [[feedback_design_system_patterns]]
2. **Borrar el demo.** "Ya no usamos el demo, solo la lib." Se fue `projects/demo` entero; el repo quedó como un único proyecto library. Consecuencia grande: **ya no hay forma de verificar nada visualmente dentro del repo**. → [[project_repo_is_library_only]]
3. **Limpieza general del repo.** Lockfiles al revés en `.gitignore`, `lib/tokens/index.ts` muerto por completo, categorías `pipes`/`utils` vacías, y la deuda de las directivas mal ubicadas. → [[feedback_lockfiles]], [[project_dead_color_tokens]], [[project_layouts_category]]
4. **Tipografía.** Primero en validation (colapsos con pérdida en su puente de tokens), lo que llevó a **extender la escala del DS** con `font-weight-black: 900` y `font-size-paragraph-xs`. Después el backoffice, que seguía en Lato con stops fijos sin 500 ni 600 — 60 declaraciones de peso que no renderizaban. → [[project_corepark_ui_tokens]], [[project_migration_status]]
5. **El backoffice, de arriba abajo.** `cp-auth-layout`, formularios de auth a componentes del DS, standalone sin NgModules, `cp-app-layout`, y finalmente `cp-app-nav` + `cp-app-nav-footer` + `cp-menu`. → [[project_migration_status]]
6. **`cp-brand`**, el lockup de marca, extraído a la lib y puesto en las tres apps. → [[project_cp_brand]]

## Lo que más costó, y por qué

**Una pregunta de Israel desbloqueó el error de fondo.** Al migrar el rail del backoffice importé **solo el chasis** y dejé dentro los componentes locales. Llevaba tres iteraciones midiendo diffs de 8px y empujando knobs (`brand-height` a 7rem, padding del footer a `spacing-6`) para que el layout del DS **imitara** al viejo. Él preguntó: *"el menú y el footer del aside, ¿son componentes en el commerce y validation?"* — sí lo eran, y con los componentes del DS dentro la geometría cuadró sola.

**La lección:** cuando algo "no se ve igual" tras adoptar un componente del sistema, la pregunta no es *¿cómo lo hago idéntico?* sino *¿qué usan las otras apps aquí?* Usar los knobs para deshacer una migración es la señal de que se está resolviendo el problema equivocado.

## Método que funcionó

- **Medir en vez de mirar.** Un decodificador PNG en Python puro para diffear capturas por regiones y sacar posiciones y colores exactos. Así se descartó que el peso de "core" fuera distinto entre apps: mismo 300 en los tres, la diferencia era el fondo.
- **CDP para lo que una captura no prueba.** Clics reales para verificar que el `cp-menu` abre y que cambiar de idioma **retraduce el nav en vivo**.
- **Probetas temporales** para ver detrás del login (bypass del guard, form deshabilitado al arrancar). Útiles pero **peligrosas**: una interrupción dejó el guard de autenticación anulado en el árbol de Israel. Si se usa una, restaurarla es lo primero al retomar.

## Verificación al cierre

Los 4 repos compilan, los 279 tests del DS pasan, y el `dist` está sincronizado a commerce, validation y backoffice (valet-web fuera por decisión). **Nada commiteado, nada publicado.**

Lo que queda abierto está en [[project_pending_work]].
