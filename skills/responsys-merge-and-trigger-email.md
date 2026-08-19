---
name: Merge a recipient into a profile list and trigger a campaign email
description: >-
  Authenticate against Responsys, upsert one or more recipients into a profile list, then
  dispatch a triggered email from an existing campaign — with the throttling, error and
  no-idempotency rules that make this safe to run unattended.
api: openapi/responsys-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/_original/responsys-openapi.json + https://docs.oracle.com/en/cloud/saas/marketing/responsys-develop/API/api.htm
operations:
  - "POST /rest/api/v1.3/auth/token"
  - "POST /rest/api/v1.3/lists/{listName}/members"
  - "GET /rest/api/v1.3/lists/{listName}/fields"
  - "POST /rest/api/v1.3/campaigns/{campaignName}/email"
---

# Merge a recipient and trigger a campaign email

## Before you start

This operation **sends real email to real people** and there is **no idempotency key**.
Treat the trigger call as at-most-once. If it times out, do not automatically retry —
verify first, or you will send twice.

## 1. Authenticate, then switch hosts

`POST /rest/api/v1.3/auth/token` against a login host —
`https://login5.responsys.net`, `https://login2.responsys.net`,
`https://login.rsys8.net`, `https://login.rsys9.net`, or the account's global-routing
host `https://{accountToken}-api.responsys.ocs.oraclecloud.com`.

Body uses one of three `auth_type` values: `password`, `token` (refresh an existing
token), or `certificate`.

The response carries `authToken` **and** `endPoint`. **Use `endPoint` as the base URL for
every subsequent call.** Continuing to call the login host is the most common integration
mistake. Send the token as `Authorization: <authToken>` — no `Bearer` prefix.

## 2. Learn the list's shape

`GET /rest/api/v1.3/lists/{listName}/fields`

Returns the field definitions of the profile list. Merge payloads must name fields that
actually exist on the list — a wrong field name comes back as `INVALID_PARAMETER` with the
offending name in `detail`.

Note that `listName` is a **human-readable name**, not an id. If someone renames the list
in the Responsys UI, this path breaks.

## 3. Merge the recipient

`POST /rest/api/v1.3/lists/{listName}/members`

Merge is an **upsert** keyed on the list's match column (typically `EMAIL_ADDRESS_` or
`CUSTOMER_ID_`), so re-merging the same record updates rather than duplicates. That is the
closest thing to idempotency this API offers — and it does not extend to step 4.

Watch out: this same path and method also serves "retrieve multiple recipients" and
"delete multiple recipients", distinguished only by the request body. Send the merge body,
not a lookup body.

The response returns a `recipientId` (RIID) per record. Keep it — RIID is the only opaque
identifier in the model and is what event notifications will quote back to you.

## 4. Trigger the email

`POST /rest/api/v1.3/campaigns/{campaignName}/email`

The campaign must already exist and be configured against this list. Optional-data and
personalization values ride in the request body.

If you need attachments, use
`POST /rest/api/v1.3/campaigns/{campaignName}/emailAttachments/actions/trigger` instead.

## Limits and errors

- Throttling is **per API function, per account, per minute**. The documented default for
  high-volume functions is **200 requests/minute**; `Merge List Recipients` is documented
  at 1000/min and `Login` at 10/min. Read the account's real ceilings from the
  unversioned `GET /rest/api/ratelimit` (no `v1.3` in that path — adding it returns 404).
- There are **no rate-limit response headers**. Nothing tells you how much quota is left.
  Budget from `/rest/api/ratelimit` up front.
- On exhaustion the body carries `errorCode: API_LIMIT_EXCEEDED`. Back off for a minute.
- `errorCode: API_BLOCKED` is **not** transient — the user is blocked from that function.
  Stop and raise a My Oracle Support request.
- `errorCode: UNEXPECTED_EXCEPTION` with detail "Not a valid authentication token" means
  re-authenticate at step 1.
- Branch on `errorCode`, never on `type` — `type` is documented as an empty string in
  every example. Full catalog: `errors/responsys-problem-types.yml`.

## Confirming the send

Responsys does not report delivery on the trigger response. Subscribe to the Event
Notification API for `EMAIL_BOUNCED`, `EMAIL_FAILED`, `EMAIL_SKIPPED` and `EMAIL_CLICKED`
to learn what actually happened — see
`skills/responsys-subscribe-to-event-notifications.md`.
