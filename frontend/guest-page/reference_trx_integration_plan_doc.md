---
name: reference_trx_integration_plan_doc
description: El artifact con el plan de integración de TRX que mantiene Jorge Valencia — fuente de verdad de pendientes y decisiones
metadata: 
  node_type: memory
  type: reference
  originSessionId: a1632e92-0e98-42a8-bf7c-9a40c171e518
  modified: 2026-09-21T22:36:06.390Z
---

**TRX Integration Plan**, de Jorge Valencia (fechado 17-sep-2026):
https://claude.ai/code/artifact/75379b97-28f6-48e9-a52b-a0151cb18e6d

Cubre las tres superficies de TRX (terminal P200 por POS Cloud, lector VP3350 por SDK móvil, y la guest page con la librería JS), las 28 APIs por consumidor con su estado, las preguntas abiertas a TRX y las decisiones tomadas.

Es la fuente de verdad para saber **qué espera a quién**. Lo que la guest page necesita de ahí: el pendiente del `ApplePay / GooglePay = "1"` en el Initialize, la decisión de hostnames de Apple Pay, y el hecho de que *"the browser submit from the guest page"* sigue sin verse.

Se lee con la herramienta de Artifacts (`action: "read"`), nunca con web-fetch. Está compartido a nivel organización.

Ver [[project_trx_integration_status]].
