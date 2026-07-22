---
name: Google Places infrastructure in frontend-commerce
description: Address autocomplete via Google Places — packages, API keys per env, referrer restrictions gotcha, provider + service + shared component. Ported from frontend-backoffice 2026-07-09.
type: reference
originSessionId: 6c66ac7f-ac13-4123-8f60-ce19ba5e1720
---
Frontend-commerce learned Google Places on 2026-07-09 for the inline "create partner" flow. Everything below was ported verbatim from `frontend-backoffice` (same versions, same keys, same pattern).

## Packages (pnpm)

```
"@googlemaps/js-api-loader": "^1.16.8"     // runtime
"@types/google.maps": "3.64.0"              // dev types
```

`google.maps` types are registered in `tsconfig.app.json` — `"types": ["google.maps"]` — otherwise the `google` global is `Cannot find name 'google'`.

## API keys (shared with backoffice, per env)

In `src/environments/environment*.ts`:

- **local + dev**: `AIzaSyCZQbxdD2KkF6WWZoqowzbm-VAE6c7D7LY`
- **prod**: `AIzaSyCtHRmXMdKoBWmw9PB8O8E0uj5rlaKDyPA`

Referenced via `import { environment } from '@env'` (alias, not `@environment`).

## HTTP referrer restrictions — the recurring 403 trap

Google Cloud Console restricts these keys by **HTTP referrer**. `frontend-commerce` dev-server runs on `https://localhost:4400 --ssl`; `frontend-backoffice` runs on `https://localhost:4100 --ssl`. Both use SSL because Google Places API refuses `http://` on non-localhost origins and it's simpler to always use https.

The keys are whitelisted only for `https://localhost:4100/*` initially. To enable commerce, add these to Website Restrictions:

- Dev key: `https://localhost:4400/*`, `https://dev-web-71f618df0c78.corepark.com/*`
- Prod key: whatever prod commerce domain resolves to

Changes take ~5 minutes to propagate. Symptom before: `403 Forbidden` from `places.googleapis.com/$rpc/google.maps.places.v1.Places/AutocompletePlaces`.

## Files ported

- `@providers/google-places-autocomplete.ts` — provides the `Loader` singleton with `libraries: ['places']`. Registered in `app.config.ts`.
- `@services/google-places-service.ts` — `getSuggestions(input)` + `getPlaceGeometry(prediction)`. Manages `AutocompleteSessionToken` per session (nulls after each `getPlaceGeometry`) to keep Google's billing quota happy.
- `@components/google-searcher/` — standalone component with `mat-autocomplete` + spinner. Emits `locationGeometry` output.
- `@definitions/location-geometry.d.ts` — `LocationGeometry` interface + `PlaceResultTypes` `const enum` (State/City/Country/PostalCode).

## Usage pattern in a dialog

Address fields (street/city/state/zipCode) start `disabled: true` in the FormGroup. On `locationGeometry` output, `.enable()` them and `patchValue({...})` with the resolved parts. Lat/lng are stored in component signals — the user never types them.

```ts
readonly form = this.#fb.nonNullable.group({
  address: this.#fb.nonNullable.control({ value: '', disabled: true }, [...]),
  city: this.#fb.nonNullable.control({ value: '', disabled: true }, [...]),
  // ...
})

onLocationGeometry(geometry: LocationGeometry): void {
  this.#latitude.set(geometry.latitude)
  this.#longitude.set(geometry.longitude)
  this.form.controls.address.enable()
  // ... enable others, patchValue, match country from ISO code
}
```

## Region filtering

`GooglePlacesService` restricts suggestions to `['us', 'ca', 'mx', 'es']` (North America + Spain) and `includedPrimaryTypes: ['street_address']`. To broaden or narrow this you edit `#fetchSuggestions` directly.
