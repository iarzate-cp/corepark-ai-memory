---
name: Local iteration flow for design-system + commerce (no publish loop)
description: How to iterate on @corepark/corepark-ui without publishing per fix. Build the lib and cp -R the dist into commerce's node_modules. Single commit + publish at feature end.
type: reference
originSessionId: ad4f5cfa-d336-49eb-bbe4-3711000eab64
---
**Setup:**
- design-system repo: `/Users/israel/Dev/design-system`
- commerce repo: `/Users/israel/Dev/frontend-commerce`
- Library published to GitHub Packages (`https://npm.pkg.github.com`), consumed as `@corepark/corepark-ui` in commerce's `package.json`.

**Dev loop while a feature is open** — avoid publishing per fix:

```bash
# 1. edit files in design-system
cd /Users/israel/Dev/design-system
# ... modify projects/corepark-ui/src/lib/...

# 2. rebuild
npm run build

# 3. sync into commerce
rm -rf /Users/israel/Dev/frontend-commerce/node_modules/@corepark/corepark-ui
cp -R dist/corepark-ui /Users/israel/Dev/frontend-commerce/node_modules/@corepark/

# 4. verify (optional)
grep -c "<some-token-that-should-be-in-the-new-bundle>" \
  /Users/israel/Dev/frontend-commerce/node_modules/@corepark/corepark-ui/fesm2022/corepark-corepark-ui.mjs
```

Commerce's dev server (`ng serve`) picks up the new bundle automatically.

**Caveats:**
- `pnpm install` in commerce will overwrite the local sync with whatever is on the registry. Re-run steps 2-3 after any dependency operation in commerce.
- The dist's `package.json` version may temporarily diverge from commerce's `package.json` version. Fine locally; bump commerce to the real version when you publish.
- Commit the design-system change ONLY when the feature is done — one commit with a version bump summarizing all lib changes, then `npm publish` from `dist/corepark-ui/`.

**When to actually publish**: when the consuming feature merges to `feature/staging` (dev deploy branch). At that point commerce's lock file needs to reference a real registry version, not the local sync.
