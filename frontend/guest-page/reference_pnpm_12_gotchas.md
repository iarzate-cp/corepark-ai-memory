---
name: reference_pnpm_12_gotchas
description: Dos fallos de pnpm 12 en la máquina de Israel y cómo se arreglan — el binario nativo que no se instala y los ajustes que se mudaron de sitio
metadata: 
  node_type: memory
  type: reference
  originSessionId: a1632e92-0e98-42a8-bf7c-9a40c171e518
  modified: 2026-09-21T22:36:14.712Z
---

En la máquina Windows de Israel (`C:/Users/Israel/DEV/CorePark`), el 2026-09-21 y tras subir a pnpm 12.4.1, dos cosas rompieron seguidas:

**1. `pnpm` deja de reconocerse como comando.** El self-update se queda sin su binario nativo: `node_modules/pnpm/pnpm` es un placeholder de shell que cmd.exe no puede ejecutar, porque el pre/postinstall (`install.js`) no corrió. Se arregla ejecutándolo a mano:

```
cd ~/AppData/Local/pnpm/.tools/pnpm/<version>_tmp_*/node_modules/pnpm && node install.js
```

Eso descarga `pnpm.exe`. Si vuelve a pasar tras otro `self-update`, es el mismo remedio.

**2. `ERR_PNPM_IGNORED_BUILDS` en cada install.** pnpm 12 ya no lee el campo `pnpm` de `package.json`, y el ajuste además cambió de nombre: **no es `onlyBuiltDependencies`, es `allowBuilds`**, y vive en `pnpm-workspace.yaml` como un mapa `paquete: true|false`. No lo escribas a mano — `pnpm approve-builds <pkg> '!<pkg-denegado>'` genera el archivo con el nombre correcto. Ojo: `-y` no es una opción válida ahí pese a aparecer en el help.
