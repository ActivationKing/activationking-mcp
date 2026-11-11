---
name: activationking
description: Use the ActivationKing MCP tools to read the user's events, guest sessions, stats and credits, to read or change event settings (kiosk screen texts, colors, timings, AI and sharing options) and white label branding, to remote-control a running booth (start, pause, resume, pick choices, fill forms, send it a photo or video), and to read or write Ops Calendar companies, event jobs and deals (for example importing a deal from a CRM as an Ops job). Use when the user mentions ActivationKing, their events or sessions, AI credits, event or kiosk settings, white label or branding, controlling or driving a booth remotely, Ops Calendar, or wants to push jobs, clients or deals into Ops.
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

## Event settings

- Call `get_settings_catalog` (with the event's app, e.g. `com.activationking.photobooth`) before any change. Use only keys from the catalog and values of the catalog type and range; never guess a key from its name.
- Read the current values with `get_event_settings` (`isSet: false` = the kiosk uses the default) and show the user the old and new value of each key before writing.
- **Confirm before `update_event_settings`.** Pass only the keys to change; `null` resets a key to its default. If any value is invalid the whole call is rejected (`invalid_setting`) and nothing is saved: fix the value and resend.
- `settings_not_supported`: the event's app has no settings catalog yet (currently only Photo Booth has one).
- The kiosk applies the change on its next sync, not instantly.

## White label

- `get_white_label` / `update_white_label` take an optional `app` (default `com.activationking.photobooth`). `Enabled`, `AppName` and `HidePromoSections` are per app; `MicrositeEnabled`, the colors and the texts are shared by all of the user's apps, so say so when changing them.
- **Confirm before `update_white_label`**, listing old and new values. Pass only the fields to change.
- Colors: `"#RRGGBB"` for solid, or `{color1, color2, color3, fillType: "Solid"|"Gradient", gradientType: "Linear"|"Radial", lineerGradientAngle}`. `null` clears a color or text.
- Logos, background media and the custom domain can't be changed through the API; tell the user to change them in the app.
- `403 white_label_not_allowed`: the user's plan doesn't include white label. Apps pick up changes within about 5 minutes.

## Remote control (driving a live booth)

- Fields whose value is `null` are left out of the replies (e.g. `sessionId`, `eventName`, `screenKey`): treat a missing field as `null`, not as an error.
- `list_remote_booths` lists the booths open for remote control (one row per booth and event): `online`, `operatorEnabled`, current `screenKey` / `screenTitle`. `eventName` can be missing; use `eventId` with `get_event` to name the event for the user.
- Only the account that runs the booth can control it. `operatorEnabled: false` means operator control is off for that event (`403 operator_disabled`); it is turned on in the booth app under Settings > Run > Remote Control > Operator control, with the event selected. `online: false` means commands and uploads fail with `409 booth_offline`; this also covers a booth app connected to another environment (for example a test build). Don't retry either; tell the user.
- **Always call `get_remote_booth` right before `send_remote_command`** and use only an action id, choice id or field option from that screen; never guess one. `screenIsLive: false` means the booth is offline and the screen is the last one it reported.
- Actions with `role: guest` press an on-screen button for the guest; `role: operator` are operator controls (pause, resume, run...). An action with a `url` is a link, not a command.
- `send_remote_command`: `actionId` from the screen; for a choice `actionId: "choice"` + `choiceId`; for a form `actionId: "submitFields"` + `fields` (`{fieldId: value}`, option fields must use one of the field's options). **Always pass `stateVersion`** from the screen you read: if the screen changed in the meantime the command is not sent and you get `accepted: false`, `stale_state` and the current screen; decide again from that screen. It returns `{accepted, reason, screen}`, where `screen` is the booth's settled screen after the action; show it to the user. Right after a run action (`op_run_*`) the returned screen can still be filling in (no title, only operator actions); if an action you expect is missing (or a command gets `action_not_available` on such a screen), wait 1-2 seconds and read the screen again with `get_remote_booth` before deciding. The `stateVersion` may not change while it fills in.
- If the choices have no labels, don't pick one yourself: tell the user and let them choose by number or on the booth.
- **Confirm before a command that starts, stops or changes a guest's session** (run, pause, a guest button) when a guest may be using the booth. Commands are not idempotent: don't resend a command just because the reply was slow; read the screen first.
- Errors before sending (nothing reaches the booth): `action_not_available` (lists the actions on the screen), `invalid_choice` (lists valid choice ids), `invalid_fields` (says which field and why). Booth rejections (`accepted: false`): `stale_state`, `action_not_available`, `not_allowed_for_role`, `screen_not_remotable` (the booth is paused; only operator actions work), `invalid_fields`.
- **Follow a session with `sessionId`:** the screen (and each `list_remote_booths` row) has `sessionId`, the running booth session. It is `null` on the welcome screen and until the booth first syncs the session (usually a few seconds), then set. Call `get_session` with it to follow the session while it runs (`status: "in_progress"`, the media so far) and to get the result photos/videos (`media`, `coverImageUrl`) once it is `completed`. `completed` is reported only once the result media is uploaded; a session whose result never uploads stays `in_progress` for 10 minutes after its last change, then shows `completed`. Older booth app versions don't fill `sessionId`; there, the last path segment of the `shareUrl` (`https://v3.activationshare.com/m/<sessionId>`) shown at the end is the session id.
- `upload_remote_media` only works while the screen has a `mediaRequest` (e.g. after the `start_upload` action); check `mediaRequest.kind` (image or video), `total` / `received` and `maxVideoSeconds` (null for photos). Pass a public `https` url or base64 data plus `fileExtension` (`.jpg`, `.png`, `.heic`, `.webp`, `.mp4`, `.mov`); photos up to 50 MB, videos up to 250 MB. Local or private-network urls are refused. The reply has `mediaId` (the stored file: uploading the same content again in the same account returns the same id; `cloudId` is the same value under its old name, not a session id), `sessionId` (the session it went to, when known) and the booth's settled `screen` after it received the file.

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
- `400 invalid_setting`: a key or value doesn't match the settings catalog / white label fields; nothing was saved.
- `400 settings_not_supported`: the event's app has no settings catalog.
- `403 operator_disabled`: operator control is off for the booth's event.
- `409 booth_offline` / `booth_moved` / `not_waiting_for_media`: the booth is offline (or connected to another environment, such as a test build), moved to another event, or isn't asking for media; read the booth again.
- `504 booth_timeout` / `502 booth_unreachable`: the booth didn't answer in time. Read the screen before trying again; the command may have run. If the screen (`stateVersion`) didn't change and a second try also times out, stop and tell the user the booth isn't responding (check that the app is open, on the current release and connected to the live service; an old `lastUpdatedAt` is a hint) instead of retrying.
- `400 invalid_file` / `wrong_media_type` / `upload_rejected`, `502 upload_failed`: the media file couldn't be used (no url or base64, bad url, content isn't a valid image or video, wrong type for the request, too large).
- `400 action_not_available` / `invalid_choice` / `invalid_fields`: the command doesn't match the current screen; the message lists what is valid. Read the screen and pick from it.
- `402`: no active subscription for the feature (e.g. the organization has no Ops Calendar subscription).
- `403 white_label_not_allowed`: the plan doesn't include white label.
- `403`: the user's role in that organization lacks the permission (e.g. Deal.View, Deal.Edit, Jobs.Create, Crm.EditBilling). Tell the user which permission is missing; don't retry.
- `409 in_use`: the company is used by a job (see archiving above).
- `409 conflict`: the record changed at the same time; read it again and retry once.
- `429`: rate limit (300 requests/min per key). Wait for the `Retry-After` seconds and avoid fetching more pages than needed.

Never print, log or ask the user to paste the API key into the conversation.
