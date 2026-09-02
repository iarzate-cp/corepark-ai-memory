---
name: feedback_signals_over_observables
description: Israel corrige el uso de observables donde ya hay signals; APIs de signals del router de Angular 21 y matchMedia en vez de resize
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 02d8be60-b4d7-4d73-99ae-4bda6564aa7f
  modified: 2026-08-31T18:14:15.168Z
---

Israel revisó código nuevo el **2026-08-31** y señaló: *"estás usando observables cuando ya tenemos signals... puedes usar `toSignal` para que todo quede en signals"*.

**Why:** el repo es signals-first (está en CLAUDE.md). Un pipeline de rxjs donde existe una fuente en signals es ruido: más imports, más superficie de suscripción y un `initialValue` que hay que adivinar.

**How to apply:** antes de escribir `toSignal(algo.pipe(...))`, comprobar si la fuente ya expone un signal.

## Lo que había que saber, y no es obvio

**El router de Angular 21 ya expone signals.** Estaba usando el patrón viejo:

```ts
// ❌ el patrón que había que sustituir
readonly #url = toSignal(
  router.events.pipe(filter(e => e instanceof NavigationEnd), map(e => e.urlAfterRedirects)),
  { initialValue: router.url }
)

// ✅
readonly #url = computed(() => router.lastSuccessfulNavigation()?.finalUrl?.toString() ?? router.url)
```

- `router.lastSuccessfulNavigation: Signal<Navigation | null>` — **esta es la correcta.**
- `router.currentNavigation: Signal<Navigation | null>` — **NO sirve para "dónde estoy"**: vuelve a `null` justo cuando se emite `NavigationEnd`, o sea que lee vacío exactamente cuando quieres saber dónde acabaste.
- `finalUrl` es el equivalente a `urlAfterRedirects`; `?? router.url` cubre la ventana previa a la primera navegación (lo que hacía el `initialValue`).
- `isActive(url, router, matchOptions): Signal<boolean>` existe (`@publicApi 21.1`) y devuelve un computed. No encajó donde la lista de rutas es un input dinámico, pero está ahí.
- `getCurrentNavigation()` está **deprecado** desde 20.2.

Truco para compartirlo sin contexto de inyección: una función normal que recibe el `Router` y devuelve el `computed` — así se puede llamar desde un inicializador de campo. Vive en `components/app-nav/app-nav-url.ts` como `currentUrl(router)`.

## Donde un observable SÍ está justificado: y aun así se puede mejorar

`resize` no tiene fuente en signals, así que `fromEvent` es legítimo. Pero en `AppLayoutState` del backoffice la pregunta real no era el ancho sino **si el rail está en pantalla**, y eso es `matchMedia`:

```ts
readonly #rail = inject(DOCUMENT).defaultView?.matchMedia('(width >= 768px)')

readonly isDesktop = toSignal(
  fromEvent<MediaQueryListEvent>(this.#rail ?? new EventTarget(), 'change').pipe(m => m.matches),
  { initialValue: this.#rail?.matches ?? true }
)
```

Tres ventajas sobre rastrear `innerWidth`: dispara **solo al cruzar el límite** (fuera el `debounceTime`), no hay ancho que guardar, y **lee su valor real de forma sincrónica** — el `initialValue: 0` anterior afirmaba "no desktop" hasta el primer resize, así que los `@defer (when isDesktop())` cambiaban de rama después de montar.

`new EventTarget()` como fuente que nunca emite evita partir el código en dos caminos por el caso sin `window`.

Ver [[feedback_angular_gotchas]] y [[project_responsive_shell]].
