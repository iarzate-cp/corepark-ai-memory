---
name: Import Paths — @corepark/corepark-ui vs subpaths
description: CLAUDE.md mentions corepark-ui/components subpaths but the actual installed package name is @corepark/corepark-ui
type: feedback
originSessionId: 68fbc8bf-6233-4778-ae34-196a53bbfd68
---
CLAUDE.md documents theoretical subpath imports (`corepark-ui/components`, `corepark-ui/tokens`) but the **actual npm package name is `@corepark/corepark-ui`**. No subpath exports are configured in the library's `package.json`. Always import from the root barrel:

```ts
import { DateRangePickerComponent } from '@corepark/corepark-ui'   // ✅
import type { PickerApplyEvent }     from '@corepark/corepark-ui'   // ✅
import { DateRangePickerComponent } from 'corepark-ui/components'   // ❌ module not found
```

**Why:** The CLAUDE.md subpath table was written aspirationally. The library's `package.json` has no `exports` map and ng-packagr doesn't generate subpath entrypoints automatically. Attempting to use subpaths fails with "Cannot find module".

**How to apply:** Always use `@corepark/corepark-ui` as the import specifier in the backoffice and any consumer. Update CLAUDE.md if/when subpath exports are actually configured.
