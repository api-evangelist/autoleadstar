---
generated: '2026-08-06'
method: generated
name: Record and verify communication consent
description: As an integrated vendor, list the Fullpath dealers you are connected to, check a contact's current consent for a channel, and write consent records back in bulk.
api: openapi/autoleadstar-fullpath-api-openapi.yml
operations: [listDealers, getConsent, storeConsents]
source: >-
  Grounded in openapi/autoleadstar-fullpath-api-openapi.yml (fetched verbatim from
  https://developers.fullpath.com/openapi.yaml, 2026-08-06). All three operationIds verified
  verbatim in the spec. Auth per authentication/autoleadstar-authentication.yml, errors per
  errors/autoleadstar-problem-types.yml, limits per rate-limits/autoleadstar-rate-limits.yml,
  tool mapping per mcp/autoleadstar-tool-crosswalk.yml.
---

# Record and verify communication consent

This is the Consent Management vendor flow — the only write path on the Fullpath API. It is a compliance ledger: you are recording whether a consumer has opted in or out of email, SMS, or phone contact at a specific dealership, and when.

## Auth
- `Authorization: Bearer <vendor API key>` — the key issued to you at consent-management provisioning, **not** a Platform JWT.
- Base URL: `https://fullpath.com/api/v2/external/consent-management`.
- See `authentication/autoleadstar-authentication.yml`.

## Steps

1. **Find your dealers** — `listDealers` (`GET /dealers`). No parameters. Returns `dealers[]` with `client_key`, `name`, `status` (`connected` | `pending`), `connected_at`. Only act on dealers whose `status` is `connected`; a `pending` dealer has not finished the integration.

2. **Check current consent** — `getConsent` (`GET /{contact_type}/consent`).
   - Path: `contact_type` — one of `email`, `phone_sms`, `phone_call`.
   - Query: `contact_value` (the raw email or phone), `client_key` (from step 1, matching `^40NM-\d+-1$`).
   - A contact with no record returns `opt_in` and `consent_timestamp` as `null`. **`null` is not `false`** — absence of a record is not an opt-out, and treating it as one will silently suppress contact.

3. **Write consent records** — `storeConsents` (`POST /{contact_type}/consents`).
   - Path: `contact_type`. Body: `client_key` plus a `contacts` array of 1–500 objects, each `{ contact_value, opt_in, consent_timestamp }`.
   - `consent_timestamp` is ISO 8601 and should be **when the consent event actually happened**, not when you called the API. This is the field a TCPA defense turns on.
   - One call is one `contact_type` for one dealer. Consent for email and SMS are separate records and separate calls.

## Notes
- **There is no idempotency key.** If a `storeConsents` call times out or returns 500, a blind retry has undefined de-duplication behavior. Read back with `getConsent` for a sample of the batch before retrying. See `conventions/autoleadstar-conventions.yml`.
- Batch to the 500 ceiling, not past it — 501 contacts is a `422`, not a partial write.
- The `client_key` space here is **disjoint** from the Platform API's integer `dealershipId`. You cannot join the two surfaces from the published contracts; carry your own mapping.
- MCP equivalents: `list_dealers`, `get_consent`, `store_consents` in `mcp/autoleadstar-mcp-tools.json`.

## Limits
- 100 requests per minute per API key. On `429`, sleep `Retry-After` seconds. `X-RateLimit-Limit` and `X-RateLimit-Remaining` appear **only on the 429**, so budget conservatively — you cannot see your remaining quota until you have already exhausted it.

## Errors
- `400` — business-rule failure outside form validation. `422` — Laravel validation, with a field-keyed `errors` map (`{"contact_value": ["The contact value field is required."]}`). `401` — bad or missing vendor key. Envelope on this surface is `{message}`, **not** the `{error, message, details}` shape the Platform API uses. See `errors/autoleadstar-problem-types.yml`.
