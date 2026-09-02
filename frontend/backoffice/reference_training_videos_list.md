---
name: Training Videos — cómo añadir videos a la lista
description: VIDEO_LIST en training-videos-page.videos.ts; los labels se derivan del título de YouTube quitando el prefijo "CorePark | <sección> |"
type: reference
originSessionId: 6e032c3e-57ae-408c-871a-8e3468d407f9
---

Israel manda URLs `https://youtu.be/<id>` pelonas ("hay que añadir esos 3 en la lista de vídeos") — sin título ni sección. Es un chore recurrente.

**Dónde:** `src/app/pages/training-videos-page/training-videos-page.videos.ts` — array `VIDEO_LIST`, entradas `{ label, youtubeId, section }` con `VideoSection.ValetApp | BackOffice`. Se agregan al final; el orden del array no importa para el render.

**Cómo resolver título y sección sin preguntar:**
```bash
curl -s "https://www.youtube.com/oembed?url=https://youtu.be/<id>&format=json" | jq -r .title
```
Los títulos vienen como `CorePark | Valet App | Smart Scan Configuration`. El `label` es **solo el último segmento** — el prefijo `CorePark | <sección> |` se descarta porque la página ya agrupa por sección. Y ese mismo segundo segmento (`Valet App` / `Backoffice`) es el que determina el `section`.

**Why:** sin el oembed hay que preguntarle a Israel el título y la sección de cada video, un round-trip por link. Con él, un chore de 3 videos se resuelve en una pasada.

**How to apply:**
- Duplicados de `label` son normales y esperados (hay tres "Employee Pin Creation" con IDs distintos) — no deduplicar ni "arreglar".
- Hay typos históricos en labels (`Backoffice Demostration`, `Residental Vehicles`). No tocarlos salvo que lo pidan: no es el scope del chore.
- Rama típica: `chore/youtube-videos` (se reutiliza entre tandas). Commit `feat(training-videos): ...`, y de ahí merge a `feature/staging` — ver [[feedback_no_git]].
