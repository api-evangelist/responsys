---
name: Find, preview and schedule a Responsys campaign
description: >-
  Locate a campaign by name or filter, render its HTML and text preview, inspect its data
  sources and proof list, then create or update its launch schedule.
api: openapi/responsys-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/_original/responsys-openapi.json
operations:
  - "GET /rest/api/v1.3/campaigns"
  - "POST /rest/api/v1.3/campaigns/actions/search"
  - "GET /rest/api/v1.3/campaigns/{campaignName}"
  - "GET /rest/api/v1.3/campaigns/{campaignName}/preview"
  - "GET /rest/api/v1.3/campaigns/{campaignName}/attributes/dataSources"
  - "GET /rest/api/v1.3/campaigns/{campaignName}/attributes/proofList"
  - "POST /rest/api/v1.3/campaigns/{campaignName}/schedule"
  - "GET /rest/api/v1.3/campaigns/{campaignName}/schedule"
  - "PUT /rest/api/v1.3/campaigns/{campaignName}/schedule/{scheduleId}"
  - "DELETE /rest/api/v1.3/campaigns/{campaignName}/schedule/{scheduleId}"
---

# Find, preview and schedule a campaign

Authenticate first (`POST /rest/api/v1.3/auth/token`, then use the returned `endPoint` and
put `authToken` in the `Authorization` header).

## 1. Find the campaign

- `GET /rest/api/v1.3/campaigns` — fetch all campaigns.
- `POST /rest/api/v1.3/campaigns/actions/search` — fetch campaigns using filters. Prefer
  this on a large account; the unfiltered collection read is cheap to call and expensive
  to page.
- `GET /rest/api/v1.3/campaigns/{campaignName}` — fetch one.

Campaigns carry `campaignStatus` with values `NONE`, `DRAFT`, `ACTIVE`, `CLOSED`. Check it
before scheduling — scheduling a `CLOSED` campaign is not a meaningful action.

`campaignName` is the address. There is no campaign id in the path.

## 2. Inspect before you commit

- `GET /rest/api/v1.3/campaigns/{campaignName}/preview` — renders the HTML and text
  bodies. This is the closest thing Responsys publishes to a dry run: **there is no
  sandbox and no test mode.** Preview and the proof list are the whole safety net.
- `GET /rest/api/v1.3/campaigns/{campaignName}/attributes/dataSources` — the tables and
  fields the campaign personalizes from. If a merge you performed did not land in one of
  these, it will not appear in the message.
- `GET /rest/api/v1.3/campaigns/{campaignName}/attributes/proofList` — the internal seed
  audience. Send to the proof list before the real one.

## 3. Schedule

- `POST /rest/api/v1.3/campaigns/{campaignName}/schedule` — create an email or push
  campaign schedule. Returns a `scheduleId`.
- `GET /rest/api/v1.3/campaigns/{campaignName}/schedule` — list schedules.
- `GET /rest/api/v1.3/campaigns/{campaignName}/schedule/{scheduleId}` — read one.
- `PUT /rest/api/v1.3/campaigns/{campaignName}/schedule/{scheduleId}` — update.
- `DELETE /rest/api/v1.3/campaigns/{campaignName}/schedule/{scheduleId}` — cancel.

`scheduleId` is one of the few real ids in the API. Store it — without it you cannot
update or cancel, and a scheduled campaign will launch.

## Rules that apply throughout

- **No idempotency key.** A retried schedule POST can create a second schedule. Read back
  with `GET .../schedule` before retrying.
- **No rate-limit headers.** Per-function, per-account, per-minute quotas; read them from
  `GET /rest/api/ratelimit` (unversioned path).
- Branch on `errorCode` in the error envelope, not on `type`. See
  `errors/responsys-problem-types.yml`.
