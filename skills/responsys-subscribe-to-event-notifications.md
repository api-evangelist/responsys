---
name: Subscribe to Responsys event notifications (webhooks)
description: >-
  Register and verify a callback URL, subscribe it to Responsys campaign event types, and
  consume the batched delivery/engagement/consent events that are the only feedback channel
  for triggered sends.
api: openapi/responsys-openapi.yml
generated: '2026-08-13'
method: generated
source: https://docs.oracle.com/en/cloud/saas/marketing/responsys-develop/API/REST/EventNotification/rest-event-notifications.htm
operations:
  - "POST /rest/api/v1.3/auth/token"
  - "GET /rest/api/v1.3/notifications/eventList"
  - "POST /rest/api/v1.3/notifications/callbacks"
  - "GET /rest/api/v1.3/notifications/callbacks/{callbackName}?action=verify"
  - "POST /rest/api/v1.3/notifications/subscriptions"
---

# Subscribe to Responsys event notifications

## Entitlement check first

The Event Notification API is **Controlled Availability**. Access is granted through a My
Oracle Support service request, and the Events tier requires an additional SKU. If these
endpoints 404 or deny, the account is not entitled — that is a commercial problem, not a
code problem.

These endpoints are **not** in the published Swagger; they are documented separately in
the Responsys Developer's Guide. The catalog of them lives in
`asyncapi/responsys-event-notification-webhooks.yml`.

## 1. Authenticate

Same two-hop flow as every Responsys call — `POST /rest/api/v1.3/auth/token`, then use the
returned `endPoint` as the base URL and the `authToken` in the `Authorization` header.

## 2. Discover what you can subscribe to

`GET /rest/api/v1.3/notifications/eventList`

Returns the 26 supported event types. They fall into three useful classes:

- **delivery-failure** — `EMAIL_BOUNCED`, `EMAIL_FAILED`, `EMAIL_SKIPPED`, `SMS_FAILED`,
  `SMS_SKIPPED`, `SMS_MO_FW_FAILED`, `MMS_SKIPPED`, `MMS_FAILED`, `PUSH_SKIPPED`,
  `PUSH_FAILED`, `PUSH_BOUNCED`, `WEBPUSH_FAILED`, `WEBPUSH_SKIPPED`, `WEBPUSH_BOUNCED`
- **engagement** — `EMAIL_CLICKED`, `WEBPUSH_CLOSED`
- **consent** — `EMAIL_OPTOUT`, `EMAIL_OPTIN`, `EMAIL_COMPLAINT`, `PUSH_OPT_IN`,
  `PUSH_OPT_OUT`, `SMS_OPT_IN`, `SMS_OPT_OUT`, `WEBPUSH_OPTIN`, `WEBPUSH_OPTOUT`
- plus `SMS_RECEIPT`

Note what is **absent**: there is no `EMAIL_SENT` and no `EMAIL_OPENED`. Do not build a
pipeline that waits for either.

## 3. Register a callback

`POST /rest/api/v1.3/notifications/callbacks`

Your endpoint must be HTTPS and reachable from Responsys. It may require HTTP Basic
authentication — supply those credentials on the callback registration.

## 4. Verify the callback

`GET /rest/api/v1.3/notifications/callbacks/{callbackName}?action=verify`

**Nothing is delivered until verification succeeds.** A silent webhook is almost always an
unverified callback.

## 5. Subscribe

`POST /rest/api/v1.3/notifications/subscriptions`

Bind the verified callback to the event types you want, and set `batchSize` (e.g. `100`).

## 6. Consume

Responsys POSTs batches of JSON to your callback. The published payload shape:

```json
{
  "campaignId": "193897761",
  "riid": "3738409741",
  "failureReason": "MAILBOX_FULL",
  "customerId": "10",
  "recipient": "example@oracle.com",
  "failureType": "EMAIL_BOUNCED",
  "channelType": "Email",
  "campaignName": "Summer Promotions"
}
```

Join `riid` back to the recipient you merged, and `campaignId` / `campaignName` back to
the campaign you triggered.

## What is not published

- **No message signature.** Authenticity rests on the verification handshake plus your own
  Basic auth. Verify the caller yourself; do not trust payload contents alone.
- **No retry policy.** Oracle does not document redelivery behaviour, so assume delivery
  is best-effort and reconcile periodically rather than relying on the stream.
- **No AsyncAPI document.** This surface is HTML-documented only.

Consume idempotently on your side: the same event may arrive more than once and there is
no event id or dedupe key in the published payload.
