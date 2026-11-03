---
name: activationking
description: Use the ActivationKing MCP tools to read the user's events, guest sessions, stats and credits, and to read or write Ops Calendar companies, event jobs and deals (for example importing a deal from a CRM as an Ops job). Use when the user mentions ActivationKing, their events or sessions, AI credits, Ops Calendar, or wants to push jobs, clients or deals into Ops.
---

# ActivationKing

The `activationking` MCP server exposes the user's own ActivationKing data. Full tool reference: https://api.activationking.com/llms-full.txt

## Start here

1. Call `get_account` first. It returns account totals and the **organizations** the user belongs to. Every `ops_*` tool needs an `organizationId` from this list.
   - If the user belongs to several organizations and hasn't said which one, ask.
2. For any Ops write, call `ops_reference_data` (for that organization) to get valid phase slugs, event and venue types, member ids (`responsibleUserIds`), product ids and deal status ids. Never invent these values.

## Reading

- Events: `list_events` (newest first; pass `nextCursor` back as `cursor` to page), `get_event`, `get_event_stats`.
- Guest visits: `list_event_sessions`, `get_session` (with result photo/video links).
- Credits: `get_credit_balance`.
- Ops: `ops_list_companies`, `ops_get_company`, `ops_list_jobs` (optional `from`/`to` in UTC), `ops_get_job`, `ops_get_deal`.

## Writing to Ops Calendar

- **Confirm before every write.** Summarize exactly what will be created, changed or deleted (organization, record, fields) and wait for the user's approval. Deletes are permanent: `ops_delete_job` also removes the job's deal, equipment assignments and comments.
- **Importing from another system (CRM, spreadsheet):** always pass `externalSource` (a stable name for the system, e.g. `"hubspot"`) and `externalId` (that system's record id) to `ops_save_company` and `ops_save_job`. Re-running the import then updates the same record instead of creating duplicates.
- Without `externalSource`/`externalId`: pass `companyId`/`jobId` to update an existing record; omitting both creates a new one. Check with `ops_list_companies` / `ops_list_jobs` before creating, to avoid duplicates.
- Only the fields you pass change. Don't send fields the user didn't ask to change.
- A new job opens a deal automatically. Set its status, currency and price lines with `ops_update_deal`. Passing `lines` **replaces all lines**, so read the deal with `ops_get_deal` first and send the full intended list.
- A company that is used by a job can't be deleted; archive it with `ops_save_company` (`isArchived: true`) instead.
- Times are UTC.

## Errors

- `401` / missing key: the `ACTIVATIONKING_API_KEY` environment variable is not set or the key was revoked. Keys are created at https://activationking.com/dashboard/api-keys.
- `402`: the organization has no Ops Calendar subscription.
- `403`: the user's role in that organization lacks the permission (e.g. Deal.View, Deal.Edit, Jobs.Create, Crm.EditBilling). Tell the user which permission is missing; don't retry.
- `409 in_use`: the company is used by a job (see archiving above).
- `429`: rate limit (300 requests/min per key). Wait for the `Retry-After` seconds and avoid fetching more pages than needed.

Never print, log or ask the user to paste the API key into the conversation.
