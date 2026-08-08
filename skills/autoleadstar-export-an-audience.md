---
generated: '2026-08-06'
method: generated
name: Export a dealership audience
description: List the saved audiences at a dealership, inspect one, and page through its shopper membership for downstream activation.
api: openapi/autoleadstar-fullpath-api-openapi.yml
operations: [listAudiences, getAudienceById, listAudienceShoppers, getShopperById]
source: >-
  Grounded in openapi/autoleadstar-fullpath-api-openapi.yml (fetched verbatim from
  https://developers.fullpath.com/openapi.yaml, 2026-08-06). All four operationIds verified
  verbatim in the spec. Relationship shape per data-model/autoleadstar-data-model.yml.
---

# Export a dealership audience

Audiences are Fullpath's saved shopper segments. This flow reads one out so you can activate it in another system.

## Auth
- `Authorization: Bearer <platform JWT>`. Base URL: `https://api.fullpath.com/v1`.

## Steps

1. **List audiences** — `listAudiences` (`GET /dealerships/{dealershipId}/audiences`). Returns `Audience` objects: `id`, `name`, `shopper_set`, `is_starred`, `is_subscribed`, `is_sales_enablement`, `is_data_enrichment`, `created_at`, `updated_at`, `updated_by`, `public_url`. Add `embed` to get `shoppers_count`.

2. **Inspect one** — `getAudienceById` (`GET /dealerships/{dealershipId}/audiences/{id}`).

3. **Page the membership** — `listAudienceShoppers` (`GET /dealerships/{dealershipId}/audiences/{id}/shoppers`) with `page` / `per_page` (max 100). Walk until `pagination.page == pagination.total_pages`. A `204` means the audience is currently empty.

4. **Deepen a member if needed** — `getShopperById` for any shopper whose single-shopper-only fields you need (`score`, `customer_tags`, `latest_sale_estimated_equity`, …).

## Notes
- **`shopper_set` is a base64-encoded JSON filter definition in an undocumented format.** You can read *who* is in an audience but you cannot reliably reconstruct *why* from the public contract. Do not attempt to reimplement the segment logic downstream — re-read the membership instead.
- `public_url` "contains authentication" per the spec. It is a capability URL: treat it as a credential, not as a link.
- The flags (`is_starred`, `is_subscribed`, `is_sales_enablement`, `is_data_enrichment`) are `integer` 1/0, not booleans. Do not coerce with a truthiness test that treats `"0"` as true.
- Membership is a point-in-time read. There are no webhooks and no event feed for audience membership changes, so re-export on a schedule — see `conventions/autoleadstar-conventions.yml`.
- MCP equivalents: `list_audiences`, `get_audience`, `get_audience_shoppers`, `get_shopper`.

## Errors
- `404` on an unknown audience id; `204` on an empty page; `401` on an expired token. Envelope `{error, message, details}`.
