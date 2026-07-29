---
name: Missing @types/crypto-js on develop — RESOLVED 2026-07-28
description: Historical: develop briefly failed pnpm start with TS2307 on crypto-js because @types/crypto-js was not added when decrypter.ts landed in the firebase-auth commit. Fixed by chore commit 61bfa0f1.
type: project
---

**Resolved 2026-07-28** in commit `61bfa0f1` on develop (`chore: add @types/crypto-js to fix TS2307 on develop build`). No action needed unless the dep gets accidentally removed again.

## What happened
`src/app/core/utils/decrypter.ts` (added in commit `a4c5b6a4` — firebase auth + live ticket monitoring) imports `crypto-js` for AES decryption. The runtime package was added to `package.json`, but no `@types/crypto-js` devDependency, so `pnpm start` on develop failed with:

```
TS2307: Cannot find module 'crypto-js' or its corresponding type declarations.
  src/app/core/utils/decrypter.ts:1:26
```

## The fix, if it ever regresses

```bash
pnpm add -D @types/crypto-js
```

Commit as `chore:` on the branch that first hits it, then merge/PR to develop. Not tied to any feature — pure dev-env unblock.
