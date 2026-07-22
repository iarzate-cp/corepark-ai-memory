---
name: Design system (@corepark/corepark-ui) edit → build → rsync → publish workflow
description: Local dev flow for making DS component changes, testing against a consumer app, publishing to the private GitHub Packages registry
type: reference
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
The design system lives at `~/Dev/design-system/`. Angular library published as `@corepark/corepark-ui` to `https://npm.pkg.github.com` (GitHub Packages, private registry).

## Repo layout

- **Source:** `projects/corepark-ui/src/lib/`
- **Build output:** `dist/corepark-ui/`
- **Library version:** `projects/corepark-ui/package.json` (separate from the workspace-level `package.json`)
- **Build script:** `npm run build` — runs `ng build corepark-ui --configuration production` + tokens compile + styles compile + path-fix sed

## Local iteration (before publishing)

Fastest edit-see-result loop when changes are not yet published:

1. Edit sources under `projects/corepark-ui/src/lib/`.
2. `npm run build` from the repo root — regenerates `dist/corepark-ui/`.
3. Rsync into the consumer's `node_modules`:

       rsync -a --delete ~/Dev/design-system/dist/corepark-ui/ \
           ~/Dev/frontend-commerce/node_modules/@corepark/corepark-ui/

4. Angular dev server picks up the change on next reload. It does not validate against `pnpm-lock.yaml` in dev mode.

## Publishing to the registry

When ready to formalize:

1. Bump `projects/corepark-ui/package.json` version (e.g., `0.0.22 → 0.0.23`).
2. `npm run build` — the dist's `package.json` inherits the new version.
3. Rsync into consumer node_modules if you want to use the new version locally immediately.
4. Publish: `cd dist/corepark-ui && npm publish` (needs GitHub Packages auth — Israel handles this).
5. In the consumer, bump the `"@corepark/corepark-ui"` version in `package.json` and run `pnpm install` to refresh `pnpm-lock.yaml` with the real integrity hash.

## Consumer's coupling

`frontend-commerce/package.json` pins the DS version. `pnpm-lock.yaml` locks the integrity hash. Between publishing and running `pnpm install` on the consumer, the lock is stale — local dev still works via the rsync, but CI would fail.

Do not manually edit the integrity hash in the lock file — it will mismatch on re-install.

## Backward compatibility rule

Prefer adding new inputs with sensible defaults over changing existing behavior. Consumers across the frontend-commerce app must keep working without changes when they pick up a new DS version.

**Example, 2026-07-02:** the `CpTableComponent` needed to support chevron-only row toggle. Solution — added `rowClickToggles = input<boolean>(true)` (default `true` preserves the whole-row-click behavior). Consumers that need chevron-only pass `[rowClickToggles]="false"`. Zero breakage for other tables using the component.

## Key components + inputs added recently (as of 2026-07-02)

- `CpTableComponent` (`0.0.23+`) — new input `rowClickToggles: boolean` (default `true`). When `false`, only the toggle cell (chevron) triggers expansion; row body clicks are inert. Keyboard support (Enter / Space) on the chevron.

## Requirements

- Node 22.x (Angular 21.2 compat).
- macOS `rsync` (default install) works fine.
- JDK version is unrelated — DS is Angular-only, no Java involved.
