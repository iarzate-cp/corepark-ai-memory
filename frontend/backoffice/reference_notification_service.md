---
name: NotificationService (corepark-ui) — cómo cablearlo
description: Toast/notification service del DS; se registra con provideNotifications({position}) y expone show({type, title, description, duration})
type: reference
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
Sistema de notificaciones toast del design system. Vive en `@corepark/corepark-ui`.

**Bootstrap:** en `app.config.ts`:
```ts
import { provideNotifications } from '@corepark/corepark-ui'

providers: [
  ...,
  provideNotifications({ position: 'bottom-right' }),
]
```

Posiciones válidas: `'top-right' | 'top-left' | 'top-center' | 'bottom-right' | 'bottom-left' | 'bottom-center'`. El host se registra solo, no hay que agregar `<cp-notification-host>` manualmente.

**Uso desde cualquier component / service:**
```ts
readonly #notifications = inject(NotificationService)

this.#notifications.show({
  type: 'success' | 'info' | 'warning' | 'error',
  title: string,          // requerido
  description?: string,   // opcional
  duration?: number,      // ms, 0 = sin auto-close
})
```

Otros métodos: `dismiss(id)`, `setPosition(pos)` (runtime).

**Convenciones de uso en el backoffice:**
- Success updates (fetch completado con nueva data): incluir contexto en la descripción — location, filtros aplicados, count de items. Título corto ("X actualizado"), descripción con detalles.
- Errores: título único ("No pudimos cargar X"), descripción específica según código del backend, y prefijar con `{{location}}` para que el usuario sepa a qué contexto aplica.
- Duración: success ~3.5s; error ~4.5s (el usuario necesita más tiempo para leer y decidir).
- Todo el copy pasa por ngx-translate — nunca hardcodear strings en el `show()`.

**Referencia adicional:** demo en `~/Dev/design-system/projects/demo/src/app/pages/notification/notification-page.ts`.
