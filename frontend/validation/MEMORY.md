# Memory Index

- [Elias Hassan docs are design reference only](feedback_elias_docs.md) — PDFs de Elias son solo diseño/UX; ignorar detalles técnicos que incluya, usar siempre el doc del backend como fuente de verdad
- [Patrón de inject — siempre field separado](feedback_inject_pattern.md) — nunca encadenar `.property` en `inject()`; siempre asignar a `readonly #name` primero
- [Acceso a arrays — usar .at()](feedback_array_at.md) — preferir `arr.at(0)` sobre `arr[0]` porque el proyecto no tiene `noUncheckedIndexedAccess` y `.at()` sí tipa como `T | undefined`
- [Valet Dashboard — estado actual e implementación](project_valet_dashboard.md) — qué está en prod, qué está oculto (Service Times), decisiones técnicas, endpoints, roles de staff, lógica dinámica de tablas
- [Design system — workflow de build y sync](reference_design_system_workflow.md) — cómo modificar corepark-ui, buildearlo y sincronizarlo con frontend-validation; cambios aplicados a staff-roster y valet-overview
- [Stations domain — endpoint, invariantes y naming en BO](reference_stations_domain.md) — `/backoffice/v1/station`, toda location tiene una default, en BO todo el CRUD vive bajo el prefijo `transfer-ticket-*`
- [Valet Service Monitoring — filtro por station](project_monitoring_station_filter.md) — decisiones no obvias del filtro (no hay "All", empty mientras carga, solo stations activas en el menú)
- [Kiosk feature — estado y decisiones no obvias](project_kiosk_feature.md) — /kiosk en frontend-validation, mergeada a develop, requiere login, Hilton-hardcoded, input a11y-warning intencional; deuda: `.claude/settings.local.json` trackeada
