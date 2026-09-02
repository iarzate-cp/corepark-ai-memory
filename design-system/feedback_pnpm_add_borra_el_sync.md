---
name: cualquier-pnpm-add-o-install-en-un-consumidor-borra-el-sync-del-ds
description: pnpm add en frontend-* revierte @corepark/corepark-ui al registro y se pierde el build local; hay que re-sincronizar siempre después
metadata: 
  node_type: memory
  type: feedback
  originSessionId: c94ef601-6615-438c-8a5a-6cefa76a8215
  modified: 2026-08-31T20:51:09.276Z
---

> **Estado 2026-08-31:** ahora mismo los tres consumidores están en `0.0.29` **instalada del registry**, así que no hay build local que perder y esto no muerde. Vuelve a aplicar en cuanto se sincronice un build sin publicar — que es el modo de trabajo normal mientras se itera.

**Cualquier `pnpm add` o `pnpm install` en un repo consumidor revierte `@corepark/corepark-ui` a la versión del registro y borra el build local sincronizado.** Visto en vivo el 2026-08-27: un `pnpm add @tabler/icons-angular` en el backoffice imprimió

```
- @corepark/corepark-ui 0.0.28
+ @corepark/corepark-ui 0.0.25 (0.0.28 is available)
```

y con eso se fueron `directives/`, los layouts nuevos y los tokens de tipografía recién añadidos.

**Why:** el sync es un `rsync` sobre el symlink que apunta al store de pnpm. pnpm no sabe nada de ese contenido; en cuanto recalcula el árbol, restaura la versión que dice el `package.json`.

**How to apply:** después de **cada** `pnpm add`/`pnpm install`/`pnpm update` en commerce, validation o backoffice, correr `pnpm run sync:<app>` desde el design-system. Y verificarlo, no asumirlo: `ls node_modules/@corepark/corepark-ui/directives` y buscar un token reciente en `tokens/tokens.css`. Ver [[project_migration_status]].

## Y el sync tampoco basta si añadiste un export nuevo: borra `.angular/cache`

Al sincronizar un build del DS que **añade un símbolo exportado**, el `ng serve` que ya estaba corriendo revienta con **página en blanco** y en consola:

```
SyntaxError: The requested module '/@fs/.../.angular/cache/.../vite/deps/@corepark_corepark-ui.js'
does not provide an export named 'BrandComponent'
```

Vite tiene los deps **pre-bundleados** en `.angular/cache` y el rsync no los invalida. **`ng build` pasa igual**, así que el fallo solo aparece en dev — y como es un error de módulo, la app no arranca y no hay nada en pantalla que apunte a la causa.

**Fix:** `rm -rf .angular/cache` y relanzar el serve (eran 39 MB en el backoffice). Solo hace falta cuando cambia la *superficie de exports*; para cambios de CSS o de implementación el sync basta.
