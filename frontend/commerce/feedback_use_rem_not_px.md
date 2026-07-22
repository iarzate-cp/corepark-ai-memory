---
name: Use REM units in SCSS — never hardcoded pixels
description: The user styles exclusively in REM. Every new SCSS should use rem (or existing rem-based tokens like --fs-*, --fw-*), never raw px values.
type: feedback
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
Always write SCSS values in **rem**, never in px.

**Why:** The user has this as a long-standing personal convention across the whole codebase. He noticed I used px in rates.component.scss (2026-07-09) and asked me to correct it going forward. It's a stylistic choice he cares enough about to flag — treat it as non-negotiable.

**How to apply:**
- Root font-size stays at the browser default 16px → `1rem = 16px` in this project
- Existing patterns to reuse:
  - Font sizes: `--fs-xs (0.625rem)`, `--fs-sm (0.75rem)`, `--fs-base (0.875rem)`, `--fs-md (1rem)`, `--fs-lg (1.25rem)`, `--fs-xl (1.375rem)`, `--fs-2xl (1.5rem)` — all defined in `src/assets/scss/_root.scss`
  - Font weights: `--fw-medium`, `--fw-semibold`, `--fw-bold`
- Conversion cheatsheet: 4px→0.25rem, 8px→0.5rem, 12px→0.75rem, 14px→0.875rem, 16px→1rem, 20px→1.25rem, 24px→1.5rem, 32px→2rem, 48px→3rem
- Exception: 1px borders can stay as `1px` (rem for hairlines looks weird — `0.0625rem` is silly). Everything else = rem.

If I catch myself typing a px value larger than 1px in a new SCSS file, stop and rewrite in rem.
