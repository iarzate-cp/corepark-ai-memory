---
name: State Classes — frontend-guest-page
description: All Signal-based state classes, their location, and purpose
type: reference
originSessionId: 5af14dee-5704-4786-bda3-2b912b2b7349
---
All in `src/app/core/states/`. All are `@Injectable({ providedIn: 'root' })`.

| Class | File | Purpose |
|---|---|---|
| `UrlParametersState` | `url-parameters/` | URL params: `uuid`, `ticket` |
| `TicketInfoState` | `ticket-info/` | Full ticket from API; computed `operatorCompanyId`, `parkingLocationId`, `headers` |
| `LoaderState` | `loader/` | Global loading spinner |
| `ColorThemeState` | `color-theme/` | `corepark` or `curbstand` theme |
| `GuestPageConfigState` | `guest-page-config/` | Per-location payment config |
| `FreedompayState` | `freedompay-state.ts` | FreedomPay sessions, payment method, tip, 15+ computed values |
| `PaymentDetailState` | `payment-detail-state.ts` | Receipt URL |
| `TicketActionsState` | `ticket-actions.ts` | Config, car request status, expected checkout |
| `RequestCarState` | `request-car.ts` | Selected minutes for car request; initialized from `ticketInfo.config.leavingIn.at(0)` |
| `GuestCfgState` | `guest-cfg-state.ts` | get-cfg response (`GuestPageCfg`): `allowRequest`, `allowPayment`, `allowMinutesRequest`, `allowDateTimeRequest`, `leavingInCfg` |
| `RecaptchaState` | `recaptcha-state.ts` | reCAPTCHA token |
| `DeviceSettingsState` | `device-settings-state.ts` | Device/platform info |
| `PlatformDetectorState` | `platform-detector-state.ts` | iOS/Android/Web detection |
| `PhoneNumberState` | `phone-number.ts` | Phone number input |
| `DialogLoaderState` | `dialog-loader-state.ts` | Dialog-scoped loader |
