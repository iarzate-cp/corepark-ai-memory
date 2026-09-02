---
name: project_phone_gate_is_survey_driven
description: The "require phone number to request car" gate has no dedicated config — it is a side effect of having an active survey on the location, managed from Commerce → Settings → Survey
metadata:
  type: project
---
Investigated 2026-09-01. There is **no** "require phone number" toggle anywhere in the product. The guest page asks for a phone number **only because the location has an active survey**.

**Chain (backend → frontend):**
1. `company.survey` row with `deactivated_at IS NULL` for that `operator_company_id` + `parking_location_id`.
2. `ms-valet-service` — `GuestPageDao.QUERY_HAS_ACTIVE_SURVEY` (`GuestPageDao.java:268`) → `hasActiveSurvey()` → `GuestPageService.java:86` sets `location.hasSurvey` on the **ticket-info** response.
3. Guest page: `PhoneNumberState.hasPhoneNumber` (`core/states/phone-number.ts:18`) → if `hasSurvey` and guest has no phone, requires 10 digits + country code; this gates Request Car and Pay buttons (`ticket-actions`, `pay-button`). Form visibility: `ticket.component.ts:101` `displayPhoneNumberSection`.
4. If ticket-info already carries `guest.phoneNumber`, the form is skipped (see [[bug_phone_number_form_gate]]).

**Who configures it today:** an operator admin in **frontend-commerce → Settings → Survey** (`/settings/survey`, `pages/settings/survey`), backed by `ms-backoffice-service` `SurveyAdminController` (`/v1/surveys`: create, PATCH, `/{uuid}/reactivate`, DELETE = soft deactivate). Creating/reactivating a survey turns the phone gate ON; deleting/deactivating turns it OFF. **frontend-backoffice has no survey UI** — Commerce is the only place.

**If a standalone "require phone" flag is wanted:** follow the hotel-info pattern ([[feature_hotel_info_gate]]) — add a column to `company.guest_page_cfg` (already holds `allow_request`, `allow_payment`, `allow_tipping`, `allow_date_time_request`, `allow_minutes_request`, `allow_hotel_room`, `allow_expected_departure`), expose it through get-cfg + `GuestPageConfig` (`frontend-commerce/core/definitions/guest-page.d.ts`) and the Commerce → Settings → Guest Page toggle screen, then read it in `PhoneNumberState` instead of `hasSurvey`.
