---
name: Project Overview — frontend-guest-page
description: Purpose, tech stack, Angular version, and key decisions for the guest-page SPA
type: project
originSessionId: 5af14dee-5704-4786-bda3-2b912b2b7349
---
Guest-facing parking payment SPA (Angular 19.2.1, version 1.8.0). Allows parking guests to look up tickets, pay, request valet car delivery, and validate tickets via QR scanner.

**Why:** CorePark's customer-facing interface for all parking locations. Supports multiple operators (CorePark, Curbstand) with per-location config from backend.

**How to apply:** Frame all feature work in terms of the guest experience (payment, car request, validation). All changes need to be aware of multi-operator/multi-gateway constraints.

## Stack
- Angular 19 standalone (no NgModules)
- Angular Signals for state (no NgRx)
- Angular Material 19
- Payment gateways: Square Web SDK, Windcave HPP, FreedomPay HPC
- Logging: Bugfender (@bugfender/sdk)
- Date: Luxon | Validation: Zod | Forms: ngx-mask
- QR: ng-qrcode + @zxing/ngx-scanner (@zxing/library and @zxing/browser are direct deps — pnpm requires explicit peer deps)
- Build: pnpm 10.27, Angular CLI 19, TypeScript 5.8.2, Node 22

## Environments
- Dev: `https://dev-web-71f618df0c78.corepark.com`
- Prod: `https://api-web.corepark.com`
- `pnpm start` → SSL dev server on port 4100
