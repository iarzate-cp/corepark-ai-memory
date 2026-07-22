---
name: "Commerce" = proyecto frontend de CorePark
description: "Commerce" no es un microservicio backend, es el proyecto frontend donde vive la UI admin/operativa
type: project
originSessionId: 7185b632-c5fd-4d35-8446-f534edde5fe4
---
**"Commerce"** es el nombre del proyecto **frontend** de CorePark, separado del backend (no está en `/Users/israel/Dev/Back-End/`).

**Why:** Aclarado por Israel el 2026-06-23. Es donde viven los módulos de UI administrativa/operativa que consumen los microservicios.

**How to apply:**
- Si Israel menciona "el commerce", "módulo X del commerce", "el front", se refiere a este proyecto frontend, NO a un microservicio backend.
- Para tareas que toquen UI nueva, el código vive en commerce; los endpoints que consume viven en uno de los 8 microservicios (típicamente ms-backoffice-service para herramientas admin).
- No buscar componentes/páginas en este directorio backend.
