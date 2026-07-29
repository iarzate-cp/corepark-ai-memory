---
name: Acceso defensivo — cadena de `?.` + `??`
description: Preferir `foo?.bar ?? default` sobre guardas tempranas cuando se lee data que puede venir undefined/null (típico en respuestas JSON parciales)
type: feedback
---

Al leer campos que pueden no venir (respuestas parciales de JSON, campos opcionales en interfaces), usar la cadena `?.` + `??` en una sola expresión en vez de `if (!x) return` seguido de accesos crudos.

**Why:** El proyecto tiene muchos campos opcionales en el ticket (`ticket.maker`, `ticket.enRoute?.origin?.parkingAreaName`, etc.) — la data llega gradualmente según el estado del ticket. El patrón `x?.y ?? default` documenta mejor la defensividad, evita ramas condicionales innecesarias y produce código más lineal.

**How to apply:**
- En utils: `const parts = employee?.trim().split(/\s+/) ?? []` antes que `if (!employee) return ''` con acceso posterior
- En templates: `{{ ticket.maker ?? '—' }}`, `{{ ticket.enRoute?.destination?.parkingAreaName }}` — nunca acceso crudo a campo opcional
- En resolvers/formatters: encadenar `x?.a?.b ?? y?.c ?? ''` como fallbacks explícitos
- El default debe reflejar la intención (string vacío, `null`, `'—'` para UI, etc.)
