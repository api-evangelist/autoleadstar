---
generated: '2026-08-06'
method: generated
name: Sync the dealership sales pipeline
description: Pull leads, appointments, tasks and activities for a dealership on a schedule and reconcile them into an external CRM or reporting warehouse.
api: openapi/autoleadstar-fullpath-api-openapi.yml
operations: [listLeads, listAppointments, listTasks, listActivities]
source: >-
  Grounded in openapi/autoleadstar-fullpath-api-openapi.yml (fetched verbatim from
  https://developers.fullpath.com/openapi.yaml, 2026-08-06). All four operationIds verified
  verbatim in the spec. Join semantics and their limits per
  data-model/autoleadstar-data-model.yml.
---

# Sync the dealership sales pipeline

The polling loop a vendor integration runs against Fullpath. There are no webhooks — every one of these is a scheduled pull.

## Auth
- `Authorization: Bearer <platform JWT>`. Base URL: `https://api.fullpath.com/v1`.

## Steps

1. **Leads** — `listLeads` (`GET /dealerships/{dealershipId}/leads`). The widest object in the API: person fields, `crm_lead_id` / `crm_customer`, vehicle-of-interest (`voi_year`, `voi_make`, `voi_model`, `voi_trim`, `voi_vin`, `voi_miles`), trade-in (`trade_in_*`), sold vehicle (`sold_vehicle_vin`, `sold_vehicle_price`), and deal economics (`front_gross`, `back_gross`, `total_gross`). Key on `crm_lead_id` when reconciling into a CRM — the Fullpath `id` is internal.

2. **Appointments** — `listAppointments` (`GET /dealerships/{dealershipId}/appointments`). `id`, `lead_id`, `lead_name`, `scheduled_at`, `is_confirmed`, `is_shown`, `department`. Join to step 1 on `lead_id`.

3. **Tasks** — `listTasks` (`GET /dealerships/{dealershipId}/tasks`). Add `embed=activities,notes,shopper` to collapse three round trips into one. Honor `deleted_at` — tasks are **soft deleted** and still returned; filter them out or your task counts will drift upward forever.

4. **Activities** — `listActivities` (`GET /dealerships/{dealershipId}/activities`). Carries both `shopper_id` and `task_id`, so it is the table that stitches work items to people.

## Join map (and where it breaks)
- `Appointment.lead_id` → `Lead.id` ✅
- `Task.shopper_id` → `Shopper.id` ✅
- `Activity.shopper_id` → `Shopper.id`, `Activity.task_id` → `Task.id` ✅
- `Lead` → `Shopper` ❌ — **the Lead object exposes no `shopper_id`.** The link only runs the other way, via `Shopper.latest_lead` / `latest_lead_date`. There is no traversal from a shopper to all of their appointments.
- Vehicle identity is flat strings on `Lead`, not the `Vehicle` schema. Do not expect `$ref` consistency; normalize on VIN yourself.

## Incremental sync
- Sort newest-first with `sort=-created_at` and stop at your last high-water mark. There is no `updated_since` filter and no delta endpoint, so **updates to older records will not be seen by a created_at watermark** — schedule a periodic full re-read for anything you need to be correct.
- `per_page` max is 100. `204` means nothing new.

## Notes
- No webhooks, no AsyncAPI, no event feed of any kind — polling is the only mechanism.
- No `429` is declared on any of these four operations and no rate limit is published for the `/v1` surface. Implement backoff anyway; see `rate-limits/autoleadstar-rate-limits.yml`.
- `total_gross`, `front_gross` and `back_gross` are the store's own deal margins. Scope any agent or downstream user you hand `listLeads` output to accordingly.
- MCP equivalents: `list_leads`, `list_appointments`, `list_tasks`, `list_activities`.

## Errors
- `400` on malformed query parameters (most often `per_page` over 100 or an unknown `sort` field), `401`, `500`. Envelope `{error, message, details}`. See `errors/autoleadstar-problem-types.yml`.
