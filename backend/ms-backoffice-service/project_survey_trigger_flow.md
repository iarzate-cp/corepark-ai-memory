---
name: Survey trigger flow — end-to-end from checkout to SMS
description: Exactly when and how a survey petition is created and sent to a customer. Triggered by POST /v2/check-out in ms-valet-service; gated purely by an active survey row in company.survey; delivered as a shortened SMS link
type: project
originSessionId: e41472d4-2583-420e-b7e9-9700af16d6ff
---
**Snapshot as of 2026-07-02.** Verified against `ms-valet-service` code — not inferred.

## Trigger endpoint

`POST /v2/check-out` on `ValetController` (`ms-valet-service`, `ValetController.java:546`) — invoked by the valet app when marking a customer's **checkout**. The trigger fires on checkout, NOT check-in. (The `check_in` column on `survey_petition` stores the earlier check-in timestamp from `parking_service`, not the moment the trigger fires.)

There is also a v1 legacy path (`ValetService.oldCheckout` line 1347) that inlines the same `handleSurvey` call. Both paths converge on `SurveyService.handleSurvey(SendSurveyRequestBean)`.

## Call chain

    ValetController.checkout()                        [ValetController.java:546]
      → ValetService.checkout(request)                [ValetService.java:1391]
        → per ticket: valetDao.checkout(...)          [line 1467]
        → if phone available:
            sendCheckoutNotification(...)             [line 1479 or 1483]
              → surveyService.handleSurvey(...)       [line 1776]
              → notificationService.sendSms(...)      [line 1796]  (Feign → ms-notifications-service)

`sendCheckoutNotification` is the single choke point in v2 — it composes SMS body = `utils.getCheckoutSms(...) + surveyLink` and sends via Feign with `SMSUsage.SMS_CHECK_OUT`.

## Preconditions (all AND-ed)

1. **Checkout completes** — `valetDao.checkout()` returns a `CheckoutResultBean` without exception.
2. **Phone available** — either DB-stored (`checkoutResponse.getPhoneCode() != null && getPhoneNumber() != null`) OR request-supplied (`request.getCustomerPhone() != null`). No phone ⇒ no SMS, no petition. Checked at `ValetService.java:1476-1484`.
3. **Active survey exists for (operator, location)** — `SurveyDao.getSurveyId()` runs:
    ```sql
    SELECT id FROM company.survey
    WHERE operator_company_id = :op
      AND parking_location_id = :loc
      AND deactivated_at IS NULL
    ```
   No active survey ⇒ `handleSurvey` returns `""` and the checkout SMS goes out **without** a survey link. Checked at `SurveyService.java:32-59`.

There is **no feature flag** — the gating is pure DB state. This means the admin CRUD we shipped is the on/off switch: creating (or reactivating) a survey in `/settings/survey` turns on the trigger for that location; deleting (soft-delete → `deactivated_at IS NOT NULL`) turns it off.

## What handleSurvey does (SurveyService.java, ms-valet-service)

1. `surveyDao.getSurveyId(op, loc)` — the gating query above.
2. `surveyDao.getSurveyPetitionSequence()` — `SELECT nextval('company.survey_petition_id_seq')`.
3. `surveyDao.getOperatorUUIDById(op)` — public UUID of the operator (for the URL).
4. `UUID surveyPetitionUUID = UUID.randomUUID()` — token per petition.
5. `surveyDao.createSurveyPetition(...)` — INSERT into `company.survey_petition` with `(id, uuid, operator_company_id, parking_location_id, ticket, check_in, phone_number, survey_id, created_at)`. SQL constant `QUERY_SAVE_SURVEY_PETITION` in `SurveyDao.java:37-42`.
6. Returns `smsService.getSurveyShortLink(operatorUUID, locationUUID, petitionUUID)` — the shortened URL.

## URL & shortening

`SmsService.getSurveyShortLink()` (`ms-valet-service`, `SmsService.java:133`):

- Full URL: `https://{survey-page.domain}/{operatorUUID}/{locationUUID}/{petitionUUID}`
- Shortener: `POST {shortener.url}/create` with header `Api-Key-ShortLink: {shortener.api-key}`, body `{"originalUrl": "..."}` — reads `shortUrl` from JSON response. If the shortener errors or the response lacks `shortUrl`, it falls back to the full URL (`SmsService.java:176-186`).

Config properties (`application.yml`, `application-{dev,prod}.yml`):

| Property | Purpose |
|---|---|
| `survey-page.domain` | Public host of `frontend-survey` (dev: `d2hu0odqnh68nk.cloudfront.net`) |
| `shortener.url` | AWS Lambda API Gateway base URL |
| `shortener.api-key` | Shared secret header for the shortener |

## Delivery

`ValetService.sendCheckoutNotification` (`ValetService.java:1786-1800`):

1. `messageBody = utils.getCheckoutSms(checkoutResponse).concat(surveyLink)` — checkout SMS text with the survey link appended (empty string if no active survey ⇒ same SMS, no link).
2. Builds `GenericSmsBean { to: "+" + phoneNumber, message, smsUsage: SMS_CHECK_OUT, ticket }`.
3. `notificationService.sendSms(smsRequest, operatorCompanyId, parkingLocationId)` — Feign call to **ms-notifications-service**. Wrapped in `try/catch (FeignException)`: SMS failure is logged but does not roll back the checkout.

**No transaction spans this.** The petition INSERT commits independently of the SMS send. If the shortener or SMS provider fails, the petition row still exists (orphaned relative to the customer never receiving the link).

## Downstream (out of scope for the trigger, listed for completeness)

- Customer opens the SMS link → `frontend-survey` renders using `petitionUUID` path segment.
- Answers POST back to **ms-backoffice-service** `POST /survey/questionnaire` (consumer endpoint, not the admin API).
- ms-backoffice marks `started_at` and inserts `survey_answer` rows.

## Tables touched by the trigger

| Table | Access | Purpose |
|---|---|---|
| `company.survey` | SELECT | Gating — active survey lookup |
| `company.operator_company` | SELECT | Public UUID for URL |
| `company.survey_petition_id_seq` | nextval | Petition PK |
| `company.survey_petition` | INSERT | The row that represents the outbound survey |
| `company.parking_service` | (implicit) | Composite FK from `survey_petition (ticket, check_in, parking_location_id, operator_company_id)` — see `project_survey_petition_lifecycle.md` |

## Key call sites (file:line)

- `ms-valet-service/.../valet/service/impl/ValetService.java:1347` — v1 inline call
- `ms-valet-service/.../valet/service/impl/ValetService.java:1776` — v2 call via `sendCheckoutNotification`
- `ms-valet-service/.../survey/service/impl/SurveyService.java:25-60` — `handleSurvey`
- `ms-valet-service/.../survey/dao/impl/SurveyDao.java:20-42` — gating + INSERT SQL
- `ms-valet-service/.../sms/service/impl/SmsService.java:133-194` — URL build + shortener

## Observability gaps (not shipped)

- No structured event when `handleSurvey` returns empty because of gating — only INFO log `"Survey petition created: {}"` on success.
- Shortener fallback (returning full URL) logs at ERROR but doesn't distinguish "sent long URL to customer" from "SMS failed" in metrics.
- No dashboard tying `SMS_CHECK_OUT` sends to `survey_petition` INSERTs — reconciling delivery vs. petition rows requires a manual join.
