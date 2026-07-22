---
name: company.partner schema quirks and the legacy create-partner endpoint
description: Column layout of company.partner, the "phone_code_id = country FK, not dial code" historical naming, existing endpoints (LocationController.createPartner + /catalogue/init), and the wrapper we built for coupons
type: reference
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
Discovered while replicating the frontend-backoffice partner-creation flow into commerce (2026-07-09). Save yourself the archaeology next time.

## `company.partner` columns (INSERT / UPDATE)

```
operator_company_id, partner_id, country_id, business_id, "name",
address, city, state, zip_code, phone_code_id, phone_number,
language_id, partner_tz, parking_location_id, latitude, longitude
```

Composite PK is `(operator_company_id, partner_id)`. `partner_id` is assigned via `MAX(partner_id) + 1` per operator (see `LocationDaoImpl.QRY_NEW_PARTNER_ID`).

**`phone_code_id` is NOT the dial code (e.g., 52, 1).** It's the FK to the country row — same value as `country_id`. Historical naming. The Country cache bean (`com.corepark.backoffice.catalogue.bean.cache.Country`) has separate fields:
- `countryId` (int, FK) — sent as `phoneCodeId` in create-partner payloads
- `countryCode` (string, "US") — sent as `countryCode`
- `phoneCode` (int, dial like 1) — display only, e.g., "US · +1" in dropdown

Credentials live in separate tables: `company.partner_credential` + `company.valet_credential`. `LocationDaoImpl.createPartner` handles the whole insert triple in one transaction.

## Existing legacy endpoint

`PUT /backoffice/location/new-partner` on `LocationController` (in `com.corepark.backoffice.location`):
- 15-field `RequestCreatePartner` — same fields as the DB columns plus `email`, `password`, `active`, `countryCode`.
- Validates phone by country (`CommonUtil.isPhoneNumberValid(phoneCodeId, phoneNumber)`).
- Resolves country from `countryCode` via `LocationDaoImpl.getCountryWithCode`.
- Resolves timezone from `latitude` + `longitude` via `commonService.getTimeZone` (Google Timezone API on backend).
- Returns legacy `Response`-shaped envelope with `partnerId` on `ResponseCreatePartner`.

## Modern wrapper we built for coupons

`PmsConnectionController` (`/crm/pms/connections`) now exposes:

- `GET /partner-init` → `ApiResponse<PartnerInitCatalogue>` with `businesses[]` (from `custom.cat_business` via cached `LocationType` list), `languages[]`, `countries[]` (each with `countryId` + `countryCode` + `phoneCode`).
- `POST /partners` → `ApiResponse<PartnerCreated>` — accepts a `CreatePartnerRequest` with modern validations, delegates to `LocationServiceImpl.createPartner`, and translates legacy `MessageCode` results to `ApiException` with `ErrorCode.PARTNER_*` (INVALID_COUNTRY / INVALID_PHONE / TIMEZONE_LOOKUP_FAILED / DUPLICATE).

Both routes stay in the modern `ApiResponse<T>` envelope so the frontend coupons module can use `notifyHttpError` uniformly.

## Catalogue init endpoint (`GET /catalogue/init`)

Legacy but still authoritative for reference data:
- `ResponseGeneralCatalogue` returns `languages`, `locationTypes` (= businesses = partner types), `countries`, `placeTypes`, `profiles`, `parkingLayouts`, `rates` in one shot.
- All lists are hydrated at startup from DB into `@Autowired List<X>` beans and refreshed lazily if empty.
- We didn't consume this directly from commerce — went through the modern `/partner-init` wrapper instead. If you need `placeTypes` or `parkingLayouts`, add them to `PartnerInitCatalogue` or create a sibling endpoint.

## The frontend gotcha

When submitting from the frontend, do:

```ts
{
  countryCode: country.countryCode,      // "US"
  phoneCodeId: country.countryId,        // the FK, NOT country.phoneCode
  // ...
}
```

The initial wrong version sent `phoneCodeId: country.phoneCode` (the dial code) — the backend accepted it because it's just a number, but validation against the FK failed and downstream lookups broke.
