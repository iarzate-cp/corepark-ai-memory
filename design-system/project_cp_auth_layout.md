---
name: cp-auth-layout-shell-de-pantallas-de-autenticaci-n
description: "API, knobs por tema y la técnica @property para animar los gradientes decorativos"
metadata: 
  node_type: memory
  type: project
  originSessionId: 492ca683-d493-4d59-950b-8911f879eff2
  modified: 2026-08-27T03:53:53.817Z
---

`lib/layouts/auth-layout/`. Extraído del `auth-layout` de **frontend-commerce**. Publicado en 0.0.28.

## API

**Inputs:** `headline` (los `\n` renderizan como salto, usa `white-space: pre-line`), `subheadline`, `brandName` ('core'), `brandAccent` ('park'), `footerText`, `visualSide` ('left'|'right'), `decorated`, `loading`, `showThemeToggle`, `themeToggleLabel`, `loadingLabel`.
**Output:** `themeToggle`. **Slots:** default (el form), `[cpAuthVisual]`.

**NO tiene input `darkMode`.** El icono del toggle se deriva de `[data-theme]` en CSS: se renderizan los dos iconos y las reglas deciden cuál se ve. Imposible desincronizar.

Como un componente ruteado no puede recibir contenido proyectado, el consumidor hace un wrapper de ~20 líneas y rutea a ése.

## Panel decorativo — todo CSS, cero imágenes

Cinco capas: color plano, retícula (2 `linear-gradient` + `mask-image`), dos halos (`radial-gradient` + `blur(5rem)` + animación `drift`), anillos concéntricos (`::before`/`::after` con `border: inherit` + `opacity`), y campo de puntos (`radial-gradient` teselado).

## Knobs por tema

| Knob | Light | Dark |
|---|---|---|
| `-visual-bg` | `primary-subtle` | `#0a1211` |
| `-visual-fg` | `text-950` | `#ffffff` |
| `-decor` | `color-mix(oklch, primary, black 48%)` | `primary` |
| `-decor-alt` | `primary-darker` | `primary-darker` |
| `-decor-grid` | 78% | 94% |
| `-decor-ring` | 70% | 88% |
| `-decor-dot` | 64% | 82% |
| `-decor-glow` | 0.48 | 0.5 |

**La lógica es inversa a la intuición:** sobre fondo claro el teal necesita MÁS opacidad; sobre fondo oscuro esos mismos valores se leen como ruido. **No hay borde divisor** entre mitades: la separación es solo de tono.

## La técnica clave: @property

`background-image` es una propiedad **discreta** — no interpola, salta. Un `transition: background` no anima los gradientes. La solución es registrar los knobs con `@property`:

```css
@property --cp-ui-auth-layout-decor {
  syntax: '<color>'; inherits: true; initial-value: transparent;
}
```

Registrados los 8 (4 `<color>`, 3 `<percentage>`, 1 `<number>`). La transición va en `:host` y **hay que listarlos uno a uno** — `transition: all` omite las custom properties registradas en algunos motores. Cross-fade de 160ms al cambiar de tema.

Soporte: Chrome 85+, Safari 16.4+, Firefox 128+. Donde no exista, las at-rules se ignoran y el tema corta en seco. Verificado que el compilador de Angular no las mangla.

**Deuda conocida:** el subtítulo usa `color-mix(visual-fg, transparent 45%)`, heredado del diseño original en dark. En light da ~3.5:1 — **por debajo de WCAG AA** (4.5:1). Arreglo: bajar a ~25-30% o usar `--cp-ui-color-text-500`.

Ver [[project_cp_app_layout]] y [[project_layouts_category]].
