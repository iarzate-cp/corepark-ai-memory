---
name: Backoffice theme — no data-theme="dark" en index.html
description: El backoffice tiene diseño mixto claro/oscuro; no aplicar data-theme="dark" globalmente
type: feedback
originSessionId: 68fbc8bf-6233-4778-ae34-196a53bbfd68
---

No añadir `data-theme="dark"` al `<html>` del backoffice (`src/index.html`).

**Why:** El backoffice tiene un diseño mixto: sidebar oscuro, pero área de contenido principal clara (blanca). Los componentes del DS que usan tokens como `--color-bg-50` renderizan en modo claro por defecto (`:root`), lo cual es correcto y consistente con el fondo blanco del contenido. Al añadir `data-theme="dark"`, componentes como `cp-table` renderizaron con fondo `#1E1E1E` creando un contraste roto contra el área blanca.

**How to apply:** Si un componente del DS necesita verse oscuro en el backoffice, aplicar los tokens manualmente en SCSS o sobreescribir las variables CSS directamente en el contexto de uso. No cambiar el tema global.
