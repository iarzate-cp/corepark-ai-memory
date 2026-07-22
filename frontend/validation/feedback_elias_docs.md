---
name: Elias Hassan docs are design reference only
description: PDFs/briefs de Elias Hassan son para contexto de diseño, no como spec técnico
type: feedback
originSessionId: f441e250-4341-49be-a54c-8b0c886f9325
---
Los documentos y PDFs de Elias Hassan (Head of Operations) son referencia de diseño y contexto operacional — no son specs técnicos. Elias no es dev, por lo que suele incluir detalles técnicos incorrectos o irrelevantes (arquitecturas de API, endpoints, WebSocket, toggles de configuración).

**Why:** En el brief de Tesla/Valet Service Monitoring, Elias describió una arquitectura REST + WebSocket + RabbitMQ que no corresponde a la implementación real (Firebase RTDB). También propuso un toggle `showEnRouteColumn` que no existe en el sistema.

**How to apply:** Cuando el usuario comparta un PDF/doc de Elias junto con un doc técnico del backend team, el doc del backend team es la fuente de verdad para implementación. Del PDF de Elias, tomar solo: diseño visual, UX propuesto, colores, layout. Ignorar: endpoints, arquitectura de datos, configs técnicas.
