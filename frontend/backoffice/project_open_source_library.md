---
name: Open-source library plan — Google Places Searcher
description: Israel wants to extract the google-searcher component into a standalone open-source Angular library to share with the community
type: project
originSessionId: b917fabb-3cc4-4d42-befa-b3beef5a45c5
---
Plan to publish the `google-searcher` component and `google-places-service` as an open-source Angular library.

**Why:** Share reusable work with the Angular community; the component solves a common problem (Places API New integration with loading UX and session token management).

**How to apply:** When working on the google-searcher component, consider parametrizing hardcoded values (region codes, primary types, debounce, min chars) and avoid tying the API surface to internal project types. Suggest extraction-friendly patterns proactively.

Candidate package name: `@corepark/angular-google-places` or similar.

Items identified for extraction:
- Decouple `LocationGeometry` from internal project definitions
- Make `includedPrimaryTypes` and `includedRegionCodes` configurable via `input()`
- Parametrize debounce duration and minimum character threshold
- Replace `ngx-translate` dependency or make i18n pluggable
- Add Storybook or standalone demo
- Publish to npm
