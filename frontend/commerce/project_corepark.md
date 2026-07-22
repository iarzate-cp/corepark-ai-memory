---
name: Corepark Frontend — Deep Project Map
description: Comprehensive architecture, modules, states, services, routing, patterns and current feature status for the Corepark parking management dashboard
type: project
originSessionId: f455ec41-24b2-4578-b7c5-531cea337a74
---
# Corepark Frontend — Project Map

**Why:** Full blueprint of the project so future sessions require zero exploration to understand where things live, what's real vs mock, and how pieces connect.

**How to apply:** Use this as ground truth before any task. Cross-reference with live files for anything that may have drifted. Treat mock-flagged modules as "UI only" — no real API calls.

---

## Identity

| Property | Value |
|---|---|
| Repo path | `/Users/israel/Dev/frontend-commerce` |
| App name | `commerce` (package.json) |
| Version | **3.4.0** (bumped 2026-06-03 — EV charging mileage goal release) |
| Angular version | **21.2.x** |
| Dev port | `4400` (HTTPS, `npm start`) |
| Test runner | **Vitest** (aligned with Angular 21+) |
| Build outputs | `dist/commerce` |

---

## Technology Stack

- **Framework:** Angular 21 · standalone components · signal-first
- **Signals:** `signal()`, `computed()`, `effect()`, `toSignal()`, `toObservable()`
- **State:** Angular Injectable services with signals — NO NgRx, NO store library
- **HTTP:** Angular `HttpClient` + functional interceptors
- **UI:** Angular Material (M2 dark theme via `mat.m2-define-dark-theme()`) + CDK
- **Charts:** Chart.js 4 via `ng2-charts`
- **Dates:** Luxon 3
- **Validation:** Zod 3
- **Auth tokens:** `@auth0/angular-jwt` (JWT decode only, NOT Auth0 platform)
- **File download:** `@clemox/ngx-file-saver`
- **Test IDs:** `uuid` npm package
- **SCSS:** Custom property tokens, two global SCSS entry points:
  - `src/styles.scss` — global resets, font, utilities
  - `src/assets/scss/material.scss` — Material overrides (light + dark mode tokens)

---

## Path Aliases (tsconfig.json)

```
@animations/*    → app/shared/animations/*
@components/*    → app/shared/components/*  (each has its own index.ts barrel)
@custom-validators/* → app/core/custom-validators/*
@definitions/*   → app/core/definitions/*
@directives/*    → app/shared/directives/*
@enums/*         → app/core/enums/*
@env             → environments/environment  (current env file)
@forms/*         → app/core/forms/*
@guards/*        → app/core/guards/*
@icons/*         → app/shared/icons/*
@init-providers/* → app/core/init-providers/*
@interceptors/*  → app/core/interceptors/*
@layouts/*       → app/shared/layouts/*
@material        → app/shared/material/material.module.ts
@mocks/*         → app/core/mocks/*
@modules/*       → app/modules/*
@pages/*         → app/pages/*
@pipes/*         → app/core/pipes/*
@providers/*     → app/core/providers/*
@services/*      → app/core/services/*
@states/*        → app/core/states/*
@stores/*        → app/core/stores/*
@utilities/*     → app/core/utilities/*
@utils/*         → app/core/utils/*
@validators/*    → app/core/validators/*
```

Import to explicit file, never barrel except `@components/*`.

---

## Environment Files

| File | Env | Endpoint |
|---|---|---|
| `environment.ts` | default/local | `https://dev-web-71f618df0c78.corepark.com/` |
| `environment.dev.ts` | dev | same dev endpoint |
| `environment.prod.ts` | prod | production URL |
| `environment.local.ts` | local | likely localhost |

All envs expose: `production`, `authorization` (Basic auth for OAuth), `endpoint`, `operatorsMediaHost`.

- **Media host:** `https://d1ewdy0bjst1f0.cloudfront.net` — logos stored as `{operatorUuid}/image.webp` or `{operatorUuid}/{locationUuid}/image.webp`

---

## App Bootstrap

**`app.config.ts`** providers:
- `provideRouter(routes)` — all routes lazy-loaded
- `provideHttpClient(withInterceptors([headersInterceptor, tokenInterceptor]))`
- `provideAnimations()`
- `MatDialogModule`, `MatSnackBarModule` via `importProvidersFrom`
- `provideNativeDateAdapter()` — Material date picker
- `provideCharts(withDefaultRegisterables())` — Chart.js
- `verifyTokenProvider` — init provider that checks JWT validity on app start
- `colorSchemeProvider` — init provider that applies dark/light mode from localStorage

---

## Routing Structure

Two root layouts:
- **`MainLayout`** — authenticated, protected by `authedGuard`
- **`AuthLayout`** — unauthenticated, protected by `unauthedGuard`

### Authenticated Routes (`MainLayout`)

| Path | Page | Notes |
|---|---|---|
| `/` | `ClientsPage` | Prospects/clients list (operators list) |
| `/pms` | `PMSPage` | Shell with router-outlet |
| `/pms/charges` | `ChargesPage` | Hotel charges table |
| `/pms/analytics` | `AnalyticsPage` | PMS statistics charts |
| `/aggregators` | `AggregatorsPage` | Shell |
| `/aggregators/overview` | `OverviewPage` | Mock |
| `/aggregators/mapping` | `MappingPage` | Shell with tabs |
| `/aggregators/mapping/ocra` | `OcraPage` | Mock |
| `/aggregators/mapping/parkchirp` | `ParkChirpPage` | Mock |
| `/aggregators/mapping/spothero` | `SpotHeroPage` | Mock |
| `/aggregators/mapping/way` | `WayPage` | Mock |
| `/aggregators/reservations` | `ReservationsPage` | Mock |
| `/aggregators/overstay-config` | `OverstayConfigPage` | Mock |
| `/settings` | `SettingsPage` | Shell, default → valet-app |
| `/settings/valet-app` | `ValetAppPage` | Real API |
| `/settings/guest-page` | `GuestPagePage` | Real API |
| `/settings/phone-number` | `PhoneNumberPage` | Real API |
| `/settings/white-label` | `WhiteLabelPage` | Real API + CloudFront |
| ~~`/settings/parking-rules`~~ | ~~`ParkingRulesPage`~~ | **RETIRED** — endpoints removed from backend (2026-05-29) |
| `/settings/screen-config` | `ScreenConfigPage` | Real API |
| `/settings/ev-charging` | `EvChargingPage` | Real API |
| `/settings/chargers` | `ChargersPage` | Real API |
| `/settings/spot-chargers` | `SpotChargersPage` | Real API |
| `/sms` | `SmsPage` | Shell |
| `/sms/analytics` | `SmsAnalyticsPage` | Mock data |
| `/sms/usage` | `SmsUsagePage` | Mock data |
| `/catalog` | `CatalogPage` | Real API — vehicle makes/models |

### Auth Routes (`AuthLayout`)

| Path | Page |
|---|---|
| `/auth/sign-in` | `SignInPage` |

---

## Authentication Flow

1. **OAuth (password grant)** — `POST /security/oauth/token` with form params (username, password, grant_type=password, type=app_commerce). Uses Basic auth header from `environment.authorization`.
2. **OTP flow** — Optional MFA:
   - `POST /security/oauth/commerce/otp/request` → challengeToken
   - `POST /security/oauth/commerce/otp/send` → sends OTP via email or SMS
   - `POST /security/oauth/commerce/otp/verify` → exchangeToken → OAuth tokens
3. **Token storage:** `localStorage['auth']` as JSON `{ access_token, refresh_token }`
4. **AuthState** reads from localStorage on init. Computed signals: `authed`, `accessToken`, `refreshToken`, `metadata`, `employeeId`.
5. **JWT decode:** `@auth0/angular-jwt` `JwtHelperService` — reads `metadata.employeeId` from decoded access token.

### Interceptors

- **`headersInterceptor`** — adds `Content-Type: application/json` (skipped for FormData) + `Authorization: Bearer {access_token}`. Skips excluded routes.
- **`tokenInterceptor`** — catches 401 → clears localStorage auth key → resets `AuthState.userTokens` to null → closes all dialogs → shows snackbar → navigates to `/auth/sign-in`.

---

## State Architecture

All states are `@Injectable({ providedIn: 'root' })` services using Angular signals. No NgRx. Pattern:

```ts
readonly data       = signal<T | null>(null)
readonly snapshot   = signal<T | null>(null)   // for saveable settings pages
```

### State Inventory

| State class | File | Purpose |
|---|---|---|
| `AuthState` | `states/auth/auth.state.ts` | JWT tokens, decoded metadata, `authed` computed |
| `CommonState` | `states/common-state.ts` | **Global operator/location selector** — persisted in sessionStorage |
| `MainLayoutState` | `states/main-layout-state.ts` | Sidebar collapse, desktop breakpoint (1280px), localStorage key `sidebarCollapsed` |
| `LoaderState` | `states/loader/loader.state.ts` | Global page loader boolean |
| `ColorSchemeState` | `states/color-scheme.ts` | dark/light mode |
| `PmsState` | `states/pms-state.ts` | PMS types, operator companies, parking locations, charges, statistics, date range |
| `ValetAppState` | `states/valet-app/valet-app-state.ts` | Valet app feature flags (4 groups, 12 flags) + savedSnapshot + getChangedFields() |
| `GuestPageState` | `states/guest-page-state.ts` | Guest page flags (1 group, 3 flags: allowRequest, allowPayment, allowTipping) |
| `WhiteLabelState` | `states/white-label-state.ts` | Operator branding (primaryColor, secondaryColor, logo) + location logos + buildLogoUrl() |
| `ScreenConfigState` | `states/screen-config-state.ts` | Screen/field visibility config + getChangedItems() |
| `EvChargingConfigState` | `states/ev-charging-config-state.ts` | EV config as `Record<string, string>` + getChangedFields() (display priorities always sent together) |
| `ChargersState` | `states/chargers-state.ts` | List of EV chargers (add/update/clear) |
| `ChargerTypesState` | `states/charger-types-state.ts` | Available charger types (cached) |
| `SpotChargersState` | `states/spot-chargers-state.ts` | Spot-charger assignments (add/update/remove/clear) |
| ~~`ParkingRulesConfigState`~~ | **DELETED** (2026-05-29) | Retired with backend endpoints |
| `AggregatorsState` | `states/aggregators/aggregators-state.ts` | **MOCK** — operators, locations, gateway mappings, overstay config |
| `SmsState` | `states/sms-state.ts` | **MOCK** — SMS records, analytics KPIs, volume timeline, cost by operator/location |
| `VehicleCatalogState` | `states/vehicle-catalog/vehicle-catalog-state.ts` | Makes list, selected make ID, models list, loadingModels flag |
| `CustomersState` | `states/customers/customers.state.ts` | Prospect clients list + selected client |
| `CountriesState` | `states/countries/countries.state.ts` | Country codes for phone number |
| `FilterByStatusState` | `states/filter-by-state/filter-by-status.state.ts` | Filter state for clients table |
| `DialogSizeState` | `states/dialog-size/dialog-size.state.ts` | Dialog size helper |
| `UuidState` | `states/uuid/uuid.state.ts` | UUID management state |
| `CatalogState` | `states/catalog/catalog.state.ts` | Catalog state (separate from vehicle-catalog) |

### CommonState — Global Context (CRITICAL)

`CommonState` is the single source of truth for which operator and location the user is operating under. It persists via sessionStorage:
- `cp_operators` — operator list cache
- `cp_selected_operator` — currently selected operator
- `cp_selected_location` — currently selected location

Most settings/EV pages read `selectedOperator()` and `selectedLocation()` from `CommonState` and pass their IDs as `Operator-Id` / `Location-Id` headers.

---

## Service Inventory

| Service | File | API base | Notes |
|---|---|---|---|
| `AuthenticationService` | `services/authentication.service.ts` | `/security/oauth/token`, `/security/oauth/commerce/otp` | OAuth + OTP flow |
| `CommonService` | `services/common-service.ts` | `crm/common/operators`, `crm/common/locations` | Operators + locations list |
| `PmsService` | `services/pms-service.ts` | `crm/pms/*` | PMS types, charges, statistics, xlsx download |
| `SettingsService` | `services/settings-service.ts` | `crm/settings/valet-app`, `crm/settings/guest-page` | Valet app + guest page config GET/PATCH |
| `WhiteLabelService` | `services/white-label-service.ts` | `crm/settings/white-label/upload-logo`, `.../delete-logo` | Logo upload (FormData to CloudFront) |
| `EvChargingConfigService` | `services/ev-charging-config-service.ts` | `backoffice/v1/config-parameters/ev-charging` | GET + PATCH config params |
| `ChargersService` | `services/chargers-service.ts` | `backoffice/v1/ev-charging/chargers` | CRUD + deactivate/reactivate. **create/update now require voltage + amperage** |
| `ChargerTypesService` | `services/charger-types-service.ts` | `backoffice/v1/ev-charging/charger-types` | GET charger types list |
| `SpotChargersService` | `services/spot-chargers-service.ts` | `backoffice/v1/ev-charging/spot-chargers` | CRUD spot-charger assignments |
| `VehicleCatalogService` | `services/vehicle-catalog-service.ts` | `backoffice/v1/vehicle-catalog/make`, `.../model`, `.../bulk` | Makes + models CRUD + bulk import |
| ~~`ParkingRulesConfigService`~~ | **DELETED** (2026-05-29) | Retired — `GET/PATCH /v1/config-parameters/parking-rules` no longer exist |
| `ScreenConfigService` | `services/screen-config-service.ts` | `backoffice/v1/screen-config/valet_app` | Screen field visibility GET/PATCH |
| `ParkingAreasService` | `services/parking-areas-service.ts` | `backoffice/parking-layout/areas` | ⚠️ **KNOWN ISSUE:** path may be missing `/v1/`; response mapped via `r?.data` (CommonResponse shape) — investigate if parking areas don't load in spot-chargers |
| `PhoneNumberService` | `services/phone-number-service.ts` | `crm/settings/phone-number` | Phone number config |
| `OtpService` | `services/otp/otp.service.ts` | `crm/otp` | OTP management (internal, not auth) |

### API URL Convention

```ts
pathServiceSetter('crm/pms')
// → `${environment.endpoint}crm/pms`
// → `https://dev-web-71f618df0c78.corepark.com/crm/pms`
```

`API_PATHS` object in `utils/path-service-setter.ts` centralizes all path strings.

### Header Convention

Most `backoffice/v1/*` endpoints require:
```http
Operator-Id: {operatorCompanyId}
Location-Id: {parkingLocationId}
```
These come from `CommonState.selectedOperator()` and `CommonState.selectedLocation()`.

---

## API Response Shapes

- **CRM endpoints:** `{ data: T, code: string, message: string }` → use `CommonResponse<T>` from `@definitions/common`
- **Backoffice endpoints:** `{ code: string, message: string, ...payload }` — each endpoint has its own named payload key (e.g. `chargers`, `spotCharger`, `parameters`)
- **EV Charging config:** `{ code, message, parameters: EvChargingConfigParameter[] }` where each param is `{ configKey, dataType, value }` as strings
- **Logo upload:** `{ code: string, data: string }` (data = path string)

---

## Feature Modules Detail

### 1. Clients (root `/`)
- Prospect clients / operators list
- `CustomersState` + clients table component
- Actions: view details, add/edit customer, decline, approve, on-trial

### 2. PMS (`/pms`)
- **Charges** — hotel room charges table with date-range filter, PMS system filter, operator/location selector. Download XLSX report.
- **Analytics** — PMS statistics charts. Filter by PMS type + date range.
- Requires selecting operator company + parking location before data loads.

### 3. Aggregators (`/aggregators`) — **ALL MOCK DATA**
- **Overview** — summary stats
- **Mapping** — dual-panel mapper for 4 aggregators: OCRA, SpotHero, ParkChirp, Way
- **Reservations** — reservation list
- **Overstay config** — overstay calculation rules per location/mapping

Uses `AggregatorsState` which initializes from `MOCK_OPERATORS`, `MOCK_LOCATIONS_BY_OPERATOR`, `MOCK_INITIAL_MAPPINGS` constants.

`AggregatorChannel` enum: `Ocra`, `SpotHero`, `ParkChirp`, `Way`.

### 4. Settings (`/settings`)

All settings pages follow the **operator → location → load config → edit → save** pattern with sticky save bar.

| Sub-route | State | Flags/Config |
|---|---|---|
| `valet-app` | `ValetAppState` | 12 boolean flags in 4 groups (Payments, Posting, Ticket transfers, Vehicle & operations) + `maxVehicleRequestCapacity` |
| `guest-page` | `GuestPageState` | 3 boolean flags (allowRequest, allowPayment, allowTipping) |
| `phone-number` | — | Phone number config |
| `white-label` | `WhiteLabelState` | Operator logo + per-location logos, color pickers. Logo → CloudFront via `WhiteLabelService`. processImage() resizes to ≤600px WebP |
| ~~`parking-rules`~~ | **RETIRED** (2026-05-29) | Flag `enforceSpotSingleOccupancy` moved to Back Office per-garage |
| `screen-config` | `ScreenConfigState` | Screen fields visibility/required toggles per screen key |
| `ev-charging` | `EvChargingConfigState` | **8 config keys** (see EV Charging section below) |
| `chargers` | `ChargersState` | EV charger CRUD — label + charger type + **voltage + amperage** (required). Deactivate/reactivate support |
| `spot-chargers` | `SpotChargersState` | Spot-charger assignment — links charger to parking spot/area |

### EV Charging Module (updated 2026-06-03)

**`/settings/ev-charging` — 4 cards:**

| Card | Settings | Keys |
|---|---|---|
| General | 1 | `ALLOW_EV_CHARGING` |
| Charging parameters | 2 | `CHARGE_CANDIDACY_THRESHOLD_PERCENT`, `CHARGE_CANDIDACY_THRESHOLD_MILES` |
| Target goal | 3 | `DEFAULT_CHARGE_TARGET_MINUTES`, `DEFAULT_CHARGE_TARGET_MILES` *(new)*, `MILES_PER_KWH` *(new)* |
| Display priority | 2 | `CURRENT_LEVEL_DISPLAY_PRIORITY_PERCENT`, `CURRENT_LEVEL_DISPLAY_PRIORITY_MILES` |

**Special rule:** display priority keys always sent together in PATCH even if only one changed.

**`/settings/chargers` — Charger form:**
- Fields: Label, Charger type (manufacturer + model only, no powerKw), Voltage (V), Amperage (A)
- Live power preview in dialog: `voltage × amperage / 1000` kW
- Table Power column shows chip: `208V · 40A · 8.32 kW` (reads `charger.powerKw`)

**Type shapes (as of v3.4.0):**
```ts
// charger-type.d.ts — powerKw REMOVED (dropped by backend in this release)
interface ChargerType { id, manufacturer, model, description }

// charger.d.ts — voltage, amperage, powerKw ADDED
interface Charger { uuid, label, voltage, amperage, powerKw, active, chargerType }

// spot-charger.d.ts — id, deactivatedAt ADDED
interface SpotCharger { id, uuid, spot, charger, parkingArea, deactivatedAt }
```

**`EvChargingConfigKey` enum — 8 values:**
```ts
AllowEvCharging, DefaultChargeTargetMinutes, ChargeCandidacyThresholdPercent,
ChargeCandidacyThresholdMiles, CurrentLevelDisplayPriorityPercent,
CurrentLevelDisplayPriorityMiles, DefaultChargeTargetMiles, MilesPerKwh
```

**Deploy dependency:** Backend DDL must be applied before deploying v3.4.0. The old frontend breaks charger creation after DDL since `voltage`/`amperage` become required.

### 5. SMS (`/sms`) — **ALL MOCK DATA**
- **Analytics** — KPI cards (total cost, total messages, active operators/locations) + cost-by-operator chart + cost-by-location donut
- **Usage** — filter by operator/location/date range, volume timeline chart, spend table, log table
- `SmsState` generates mock records on instantiation using `generateMockRecords()`.

### 6. Catalog (`/catalog`)
- Vehicle makes + models management
- Two-panel: makes table (left) → models table (right, shows on make select)
- CRUD for makes (rename, delete) and models (add, edit, delete)
- Bulk import via JSON upload dialog
- `VehicleCatalogService` → `backoffice/v1/vehicle-catalog/make`, `/model`, `/bulk`

---

## Shared Components Inventory (key ones)

| Component | Path | Purpose |
|---|---|---|
| `SearchableDropdown` | `shared/components/searchable-dropdown` | Searchable dropdown with `DropdownItem[]` input. Used for operator/location selectors across settings pages |
| `StickySaveBar` | `shared/components/sticky-save-bar` | Floating save/discard bar for settings pages |
| `ToggleGroupSection` | `shared/components/toggle-group-section` | Renders `FlagGroup` with `MatSlideToggle` items |
| `ConfirmDialog` | `shared/components/confirm-dialog` | Generic confirm/destructive dialog |
| `ChargerFormDialog` | `shared/components/charger-form-dialog` | Create/edit charger — label + type + **voltage + amperage** + live powerKw preview |
| `SpotChargerFormDialog` | `shared/components/spot-charger-form-dialog` | Assign charger to spot |
| `StatCard` | `shared/components/stat-card` | KPI card with label, value, trend badge |
| `DualPanelMapper` | `shared/components/dual-panel-mapper` | Aggregators mapping interface |
| `PmsTable` | `shared/components/pms-table` | PMS charges data table |
| `PmsFilter` | `shared/components/pms-filter` | Date range + operator/location filter |
| `PmsCard`, `PmsCards` | `shared/components/pms-card/cards` | PMS KPI cards |
| `PmsChart` | `shared/components/pms-chart` | Chart.js chart wrapper |
| `VehicleMakesTable` | `shared/components/vehicle-makes-table` | Makes table with actions |
| `VehicleModelsTable` | `shared/components/vehicle-models-table` | Models table with actions |
| `BulkImportDialog` | `shared/components/bulk-import-dialog` | JSON bulk import for vehicle catalog |
| `RenameMakeDialog` | `shared/components/rename-make-dialog` | Inline rename dialog |
| `ClientsTable` | `shared/components/clients-table` | Main operators/clients table |
| `Loader` | `shared/components/loader` | Global spinner |
| `PageLabel` | `shared/components/page-label` | Page title display |
| `WhiteLabelPreview` | `shared/components/white-label-preview` | Preview of branding colors |
| `ScreenFieldsSection` | `shared/components/screen-fields-section` | Screen config field toggles |
| `SmsKpiCards`, `SmsCostCharts`, `SmsVolumeChart`, `SmsLogTable`, `SmsSpendTable` | `shared/components/sms-*` | SMS analytics components |

---

## Shared Directives

| Directive | Purpose |
|---|---|
| `ActionerDirective` | Pill-shaped uppercase button for inline table actions |
| `RippleDirective` | Material ripple effect on clickable elements |
| `SearcherDirective` | Search input behavior |

---

## Layouts

- **`MainLayout`** (`shared/layouts/main-layout`) — sidebar nav + router-outlet. `MainLayoutState` controls sidebar collapse. Global loader driven by `LoaderState`.
- **`AuthLayout`** (`shared/layouts/auth-layout`) — centered auth container.

### Navigation Menu (sidebar)

Defined in `main-layout.config.ts`. Sections:
1. **Operators** (root `/`)
2. **PMS** — Charges, Analytics
3. **Aggregators** *(preview)* — Overview, Mapping (Ocra/ParkChirp/SpotHero/Way), Reservations, Overstay config
4. **Settings** — Valet app, Guest page, Phone number, White label, Screen config, EV Charging, Chargers, Spot chargers, Damage config
5. **SMS** *(preview)* — Analytics, Usage
6. **Catalog** — Vehicle catalog

Icons from `shared/icons/`: all Lucide (outlined). Each icon is a standalone component.

---

## Utility Functions (key ones)

| Utility | File | Purpose |
|---|---|---|
| `pathServiceSetter(path)` | `utils/path-service-setter.ts` | Prepends `environment.endpoint`. Also exports `API_PATHS` const object |
| `processImage(file)` | `utils/image-processor.ts` | Resize to ≤600px wide, convert to WebP 85% quality, returns `Promise<Blob>` |
| `setLocalStorage/getLocalStorage` | `utils/storage.ts` | Typed localStorage wrappers. STORAGE_KEYS: `auth`, `colorScheme`, `sidebarCollapsed` |
| `setMatDialogConfig<T>(options)` | `utils/dialog.ts` | Wrapper for `MatDialogConfig` |
| `setErrorSnackBar(err)` | `utils/snack-bar.ts` | Maps HTTP error to snackbar config |
| `successSnackBar` | `utils/snack-bar.ts` | Shared success snackbar config |
| `color-scheme.ts` | `utils/color-scheme.ts` | Toggle dark/light mode via `data-theme` attribute on `<html>` |

---

## Enums

| Enum | File | Values |
|---|---|---|
| `ColorScheme` | `enums/color-scheme.ts` | `Dark`, `Light` |
| `AggregatorChannel` | `enums/aggregator-channel.ts` | `Ocra`, `SpotHero`, `ParkChirp`, `Way` |
| `EvChargingConfigKey` | `enums/ev-charging-config-key.ts` | 8 keys: AllowEvCharging, DefaultChargeTargetMinutes, ChargeCandidacyThresholdPercent/Miles, CurrentLevelDisplayPriorityPercent/Miles, **DefaultChargeTargetMiles, MilesPerKwh** |
| ~~`ParkingRulesConfigKey`~~ | **DELETED** (2026-05-29) | Retired with the module |
| `PmsStatus` | `enums/pms-statis.ts` | PMS status codes |
| `VehicleCatalogErrors` | `enums/vehicle-catalog-errors.ts` | Error codes for bulk import |
| `OverstayCalcMethod` | `enums/overstay-calc-method.ts` | Overstay calculation methods |
| `CustomersService` | `enums/customers-service.ts` | Customers service codes |

---

## Type Definitions (`.d.ts` files)

All in `core/definitions/`:
- `auth.d.ts` — `Tokens`, `OAuthResponse`, `OtpChallengeResponse`, `OtpSentResponse`, `DecodedToken`
- `pms.d.ts` — `PmsType`, `OperatorCompany`, `ParkingLocation`, `PmsCharges`, `FilterDateRange`, `PmsStatistics`
- `valet-app.d.ts` — `ValetAppConfig`, `FlagGroup`, `FeatureFlag`
- `guest-page.d.ts` — `GuestPageConfig`
- `white-label.d.ts` — `OperatorBranding`, `LocationLogo`
- `ev-charging-config.d.ts` — `EvChargingConfigParameter`, `EvChargingConfigResponse`
- `charger.d.ts` — `Charger` *(+voltage, +amperage, +powerKw as of v3.4.0)*, `ChargerResponse`, `ChargersResponse`
- `charger-type.d.ts` — `ChargerType` *(-powerKw removed in v3.4.0)*, `ChargerTypesResponse`
- `spot-charger.d.ts` — `SpotCharger` *(+id, +deactivatedAt as of v3.4.0)*, `SpotChargerParkingArea`, `SpotChargerResponse`, `SpotChargersResponse`
- `vehicle-catalog.d.ts` — `VehicleMake`, `VehicleModel`, `BulkMakeInput`, `BulkImportResponse`
- `aggregators.d.ts` — `GatewayMapping`, `MockOperator`, `MockLocation`, `OverstayConfig`, `LocationOverstayRow`
- `sms.d.ts` — SMS analytics types
- `screen-config.d.ts` — `ScreenConfigScreen`, `ScreenConfigPatchItem`
- `common.d.ts` — `CommonResponse<T>`, `OperatorItem`, `LocationItem`
- `parking-area.d.ts` — `ParkingArea`
- `customers-v2.d.ts` — `ProspectClient`, `ProspectClients`
- `http.d.ts` — `SuccessResponse`

---

## Settings Pages — Canonical Pattern

Every settings page follows this exact flow:

1. **Constructor:** Call `#restoreSelection()` — if operator/location already in `CommonState`, reload their config.
2. **Operator pick:** User selects operator → `commonState.selectOperator()` → loads locations → clears current config state.
3. **Location pick:** User selects location → `commonState.selectLocation()` → `#loadConfig(opId, locId)`.
4. **Load config:** Call service GET → map response → `state.configData.set(map)` AND `state.savedSnapshot.set({ ...map })`.
5. **Edit:** User toggles/inputs → update only `state.configData` (NOT snapshot).
6. **Dirty check:** `computed(() => JSON.stringify(configData()) !== JSON.stringify(savedSnapshot()))`.
7. **Save:** `state.getChangedFields(current, saved)` → PATCH service → on success: `state.savedSnapshot.set({ ...current })` → snackbar.
8. **Discard:** `state.configData.set({ ...savedSnapshot() })`.
9. **StickySaveBar:** Shown when `hasChanges()` is true. Emits `saved` and `discarded`.

---

## Current Active Branch

**`feature/ev-charging-milage-goal`** (as of 2026-06-03) — mileage goal support for EV charging. Adds voltage/amperage to chargers, two new config keys, updated UI grouping.

---

## Known Open Issues

- **`ParkingAreasService`** (`backoffice/parking-layout/areas`) — parking areas not loading in spot-chargers page. Suspected causes: (1) path missing `/v1/` vs all other backoffice endpoints, (2) response mapped via `r?.data` (CRM shape) but backoffice may return a different envelope. Needs investigation with backend team to confirm correct URL and response shape.

---

## Mock vs Real Data Summary

| Module | Data source |
|---|---|
| Clients / Operators | Real API |
| PMS (charges, analytics) | Real API |
| Settings (all sub-routes) | Real API |
| Catalog (makes/models) | Real API |
| Aggregators (all) | **MOCK** — `AggregatorsState` with hardcoded data |
| SMS (analytics, usage) | **MOCK** — `SmsState` generates random records on init |

---

## SCSS Architecture

Global SCSS partials in `src/assets/scss/`:
- `_root.scss` — CSS custom property tokens (both `:root` and `[data-theme="dark"]`)
- `_vars.scss` — SCSS variables
- `_buttons.scss` — `.ui-button` + variants (--secondary, --destructive)
- `_actioner.scss` — `actioner` directive pill styles
- `_cards.scss` — card base styles
- `_tables.scss` — table base styles
- `_inputs.scss` — input overrides
- `_dialogs.scss`, `_custom-mat-dialog.scss` — dialog styles
- `_snackbars.scss` — `snackbar-success`, `snackbar-error`, `snackbar-warning` panel classes
- `_breakpoints.scss`, `_media-queries.scss` — responsive mixins (Small, Medium, Laptop)
- `material.scss` — Material M2 dark theme overrides for light mode compatibility
- `_font.scss`, `_main-font.scss` — Inter font
- `_spinner.scss` — global loader

---

## LocalStorage Keys

| Key | Value | Managed by |
|---|---|---|
| `auth` | `{ access_token, refresh_token }` JSON | `AuthState`, `AuthenticationService` |
| `colorScheme` | `'dark'` or `'light'` | `ColorSchemeState`, `colorSchemeProvider` |
| `sidebarCollapsed` | `true` or `false` | `MainLayoutState` |

## SessionStorage Keys

| Key | Value | Managed by |
|---|---|---|
| `cp_operators` | `OperatorItem[]` JSON | `CommonState` |
| `cp_selected_operator` | `OperatorItem` JSON | `CommonState` |
| `cp_selected_location` | `LocationItem` JSON | `CommonState` |
