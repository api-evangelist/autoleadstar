---
generated: '2026-08-06'
method: generated
name: Build a shopper 360 profile
description: Resolve a dealership shopper from an email or phone number, then assemble the full profile — contacts, audience memberships, and the behavioral event stream.
api: openapi/autoleadstar-fullpath-api-openapi.yml
operations: [listShoppers, getShopperById, listShopperEmails, listShopperPhones, listShopperAudiences, listShopperEvents]
source: >-
  Grounded in openapi/autoleadstar-fullpath-api-openapi.yml (fetched verbatim from
  https://developers.fullpath.com/openapi.yaml, 2026-08-06). All six operationIds verified
  verbatim in the spec. Entity graph per data-model/autoleadstar-data-model.yml; pagination
  and embed semantics per conventions/autoleadstar-conventions.yml.
---

# Build a shopper 360 profile

Turn a contact identifier you already hold into the complete Fullpath CDP view of that person at one dealership.

## Auth
- `Authorization: Bearer <platform JWT>`.
- Base URL: `https://api.fullpath.com/v1`. Staging: `https://staging-api.fullpath.com/v1`.
- Every operation is scoped to a `dealershipId` path parameter. There is **no** discovery endpoint for dealership ids — you must be told which ones your token covers.

## Steps

1. **Resolve the shopper** — `listShoppers` (`GET /dealerships/{dealershipId}/shoppers`).
   - `q` is an **exact** match on email address or phone number in E.164 form. It is not fuzzy; normalize the phone number before you send it.
   - Add `embed=primary_contact,latest_lead_date,latest_sale_date` to avoid a second round trip for the common fields.
   - Paginate with `page` / `per_page` (max 100). Sort with `sort=-created_at`.
   - **A `204` means no match**, with no body at all — do not try to parse JSON.

2. **Fetch the full record** — `getShopperById` (`GET /dealerships/{dealershipId}/shoppers/{id}`). This is the only place ten of the fields exist: `score`, `public_link`, `customer_tags`, `is_enriched`, `salesperson`, `group_shopper_id`, `financial_durability_index`, `aim_propensity_score`, `household_flag`, `latest_sale_estimated_equity`. They never appear on the list endpoint regardless of `embed`.

3. **Enumerate contact channels** — `listShopperEmails` (`GET .../shoppers/{id}/emails`) and `listShopperPhones` (`GET .../shoppers/{id}/phones`). A shopper commonly has several of each; the `primary_contact` embed shows only one.

4. **Read segment membership** — `listShopperAudiences` (`GET .../shoppers/{id}/audiences`). Tells you which saved segments this person currently falls into.

5. **Pull the behavior** — `listShopperEvents` (`GET .../shoppers/{id}/events`). This is the payload that makes the profile a 360 rather than a contact card: a `oneOf` over twenty event types — `PageViewEvent`, `LeadEvent`, `SaleEvent`, `ConversionEvent`, `InputTrackingEvent`, `AdClickEvent`, `EnrichmentEvent`, `LeadHandlingSMSSentEvent`, `EmailClickEvent`, `EmailOpenEvent`, `EmailSentEvent`, `SMSClickEvent`, `SMSRepliedEvent`, `SMSSentEvent`, `AppointmentEvent`, `ServiceLeadEvent`, `ServiceROEvent`, `ServiceAppointmentEvent`. Discriminate on the event's own `type` before reading variant-specific fields.

## Notes
- **This operation has no MCP tool.** If you are working through `mcp/autoleadstar-mcp-tools.json`, steps 3 and 5 are unavailable — the agent surface stops at identity and segment membership. See `mcp/autoleadstar-tool-crosswalk.yml`.
- `public_link` on a shopper and `public_url` on an audience are **capability URLs that carry authentication**. Never log them, never persist them in an agent context, never forward them.
- The scoring fields (`score`, `marketing_engagement_score`, `loyalty_score`, `financial_durability_index`, `aim_propensity_score`) are undocumented model outputs about a named consumer. Treat them as inferred personal data.
- Pagination is page-number based with no cursor. Deep walks over a large shopper base are not stable against concurrent writes.

## Errors
- `404` — no such shopper for that `dealershipId`; shopper ids are dealership-scoped.
- `204` — empty result on any list step.
- `401` — expired JWT. Note that `api.fullpath.com` returns `401` for **every** path including nonexistent ones, so a `401` never confirms the endpoint you called exists.
- Envelope is `{error, message, details}`. See `errors/autoleadstar-problem-types.yml`.
