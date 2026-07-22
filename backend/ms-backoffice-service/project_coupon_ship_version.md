---
name: Coupon feature — frontend-commerce ship version is 3.6.0
description: The Coupons feature (feature/coupons branch across BE + FE) ships to prod as frontend-commerce v3.6.0. Track this for the version-bump PR
type: project
originSessionId: 389cf61a-37a2-48a8-b1d5-8fd8e7808e3c
---
The coupon management feature will land in production as **frontend-commerce v3.6.0**.

**Why:** Israel confirmed this on 2026-07-08 as the target release version once feature/coupons merges. Ship scope includes: BE endpoints (bulk create/append, list summaries, delete-when-no-USED, export XLSX, available rates) + FE `/settings/coupons` page (table view with type-to-confirm delete, dialog for new + add-more, rate restrictions multi-select).

**How to apply:** When preparing the frontend-commerce release PR for `feature/coupons` → `main`, the accompanying version bump in `package.json` (and any changelog) should target `3.6.0`. Do not silently invent a different version. If a hotfix goes out first and consumes 3.6.0, coordinate with Israel to re-anchor before the coupon PR merges.
