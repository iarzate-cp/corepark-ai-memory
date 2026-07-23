---
name: CP color tokens — usar --color-main-* (teal), NO --color-primary-* (azul)
description: El teal de marca Corepark vive en --color-main-500, no en --color-primary-500 (que es azul y el usuario no lo quiere)
type: project
originSessionId: 85943526-dc15-4804-8e1b-d41f97bb9dcf
---
Los tokens de color del backoffice están en `src/assets/scss/_root.scss`:

- `--color-main-*` (100–700): **teal Corepark**, `hsl(178, ...)`. Es el brand color. `--color-main-500` = `hsl(178, 100%, 34%)`.
- `--color-primary-*` (100–700): **azul**, `hsl(220, ...)`. El usuario explícitamente NO lo quiere en features nuevas — "hay un azul que no me gusta".
- `--color-accent-*` (300–700): amarillo (`hsl(49, ...)`).
- `--color-warn-*` (300–700): rojo/warning (`hsl(10, ...)`).
- `--color-grey-*` (100–700, más `--color-grey`): escala de grises para bordes, backgrounds, texto muted.

**Why:** Al aplicar estilos en Activity by Rate Class usé `--color-primary-500` para active states y focus — el usuario reportó "hay un azul que no me gusta" y pidió usar el color base de `_root.scss` (el teal `--color-main-*`).

**How to apply:**
- Active states / focus outlines / brand accents: `--color-main-500` (o 600/700 para variantes más oscuras).
- Hover suave sobre elementos con brand: `--color-main-100` + `--color-main-700` para texto.
- **No usar `--color-primary-*` en features nuevas** — es azul y no matchea el brand actual.
- Tipografía: `--fs-small` / `--fs-medium` / `--fs-regular` / `--fs-big` (NO `--fs-sm` / `--fs-md` — esos son de otros DS).
- Weights: `--fw-thin` / `--fw-light` / `--fw-regular` / `--fw-bold` / `--fw-black`. **No hay `--fw-medium`** — usar `--fw-bold` o `--fw-regular`.
