---
name: Preferencia de conteos verificados por grep
description: El usuario prefiere conteos exactos verificados en código, no estimaciones de documentación
type: feedback
originSessionId: fc4c993d-0479-4108-8ad1-e63c64662c52
---
Usar grep sobre el código fuente para contar endpoints, no estimaciones basadas en documentación o análisis previo.

**Why:** En esta sesión los conteos estimados (222–280) estaban muy por debajo del real (459). La documentación de backoffice tenía solo 11 de 60+ controllers documentados.

**How to apply:** Cuando se pida contar endpoints u otros elementos del código, siempre correr grep primero:
```bash
grep -rl --include="*.java" -E "@(GetMapping|PostMapping|PutMapping|DeleteMapping|PatchMapping)" \
  <proyecto>/src/main | grep -v "/feign/" | grep -iv "feignservice|feignclient" \
  | xargs grep -h -E "@(GetMapping|PostMapping|PutMapping|DeleteMapping|PatchMapping)" | wc -l
```
Excluir siempre los Feign client interfaces (directorio `/feign/` o archivos `*FeignService`).
