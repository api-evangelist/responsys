---
name: Manage Responsys profile lists, extensions and supplemental tables
description: >-
  Work the Responsys data layer — create and inspect profile lists, look recipients up by
  RIID or query attribute, extend profiles with PET columns, and merge non-audience data
  into supplemental tables.
api: openapi/responsys-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/_original/responsys-openapi.json
operations:
  - "GET /rest/api/v1.3/lists"
  - "POST /rest/api/v1.3/lists"
  - "GET /rest/api/v1.3/lists/{listName}/fields"
  - "POST /rest/api/v1.3/lists/{listName}/members"
  - "GET /rest/api/v1.3/lists/{listName}/members"
  - "GET /rest/api/v1.3/lists/{listName}/members/{riid}"
  - "GET /rest/api/v1.3/lists/{listName}/members/count"
  - "DELETE /rest/api/v1.3/lists/{listName}/members/{riid}"
  - "GET /rest/api/v1.3/lists/{listName}/listExtensions"
  - "POST /rest/api/v1.3/lists/{listName}/listExtensions"
  - "POST /rest/api/v1.3/lists/{listName}/listExtensions/{petName}/members"
  - "GET /rest/api/v1.3/lists/{listName}/listExtensions/{petName}/members/{riid}"
  - "GET /rest/api/v1.3/suppData"
  - "POST /rest/api/v1.3/folders/{folderName}/suppData/{tableName}/members"
---

# Manage Responsys profile data

Authenticate first (`POST /rest/api/v1.3/auth/token`), then call the `endPoint` returned in
the auth response with `Authorization: <authToken>`.

## The shape of the data layer

Three tiers, and they are not interchangeable:

1. **Profile list** — the audience. One row per person. `GET /rest/api/v1.3/lists`.
2. **Profile extension table (PET)** — extra columns hung off a profile list, joined on
   RIID. `GET /rest/api/v1.3/lists/{listName}/listExtensions`.
3. **Supplemental table** — relational data that is *not* an audience (catalog rows,
   transactions), stored under a folder. `GET /rest/api/v1.3/suppData`.

Everything is addressed by **name** — `listName`, `petName`, `tableName`, `folderName`.
The one opaque identifier is **RIID**, the recipient id.

## Profile lists

- `GET /rest/api/v1.3/lists` — all lists.
- `POST /rest/api/v1.3/lists` — create a list, with or without brand context.
- `GET /rest/api/v1.3/lists/{listName}/fields` — field definitions. Call this before any
  merge; a field name that does not exist returns `INVALID_PARAMETER`.

## Recipients

- `POST /rest/api/v1.3/lists/{listName}/members` — merge (upsert) recipients. Returns RIIDs.
- `GET /rest/api/v1.3/lists/{listName}/members` — look up by query attribute (e.g. email).
- `GET /rest/api/v1.3/lists/{listName}/members/{riid}` — look up by RIID.
- `GET /rest/api/v1.3/lists/{listName}/members/count` — count matches for a query attribute.
  Use this before paging a large result.
- `DELETE /rest/api/v1.3/lists/{listName}/members/{riid}` — delete one.
- `DELETE /rest/api/v1.3/lists/{listName}/members` — delete by query attribute.

**The overload trap.** `POST /rest/api/v1.3/lists/{listName}/members` serves *three*
different intents — merge, retrieve-multiple, and delete-multiple — selected by the
request body, not by the path or method. You cannot tell from the URL what a call will do.
Build the body deliberately and assert on it before sending; a malformed "retrieve" body
should never be able to become a delete.

## Profile extensions

- `POST /rest/api/v1.3/lists/{listName}/listExtensions` — create a PET.
- `POST /rest/api/v1.3/lists/{listName}/listExtensions/{petName}/members` — merge PET rows
  (same three-way overload as above).
- `GET /rest/api/v1.3/lists/{listName}/listExtensions/{petName}/members/{riid}` — read one
  row by RIID.
- `GET /rest/api/v1.3/lists/{listName}/listExtensions/{petName}/members` — read by query
  attribute; supports `sdate`/`edate` for date-range filtering.

A PET row belongs to a profile list recipient via RIID. Merge the recipient first.

## Supplemental tables

- `GET /rest/api/v1.3/suppData` — all supplemental tables.
- `POST /rest/api/v1.3/folders/{folderName}/suppData` — create one under a folder.
- `POST /rest/api/v1.3/folders/{folderName}/suppData/{tableName}/members` — merge records
  (with primary key).
- `GET /rest/api/v1.3/folder/{folderName}/suppData/{suppTableName}` — fetch all fields or
  just the primary-key fields. Note the singular `folder` in that path — it is not a typo
  in this document, it is what the published spec declares.

## Operating rules

- Merge is an upsert on the match key, so re-merging the same record is safe. Nothing else
  here is — there is no idempotency key.
- `Merge List Recipients` is documented at 1000 requests/minute, `Retrieve List Recipients`
  at 100. Read your account's real ceilings from `GET /rest/api/ratelimit`.
- No rate-limit headers come back on any response. See
  `rate-limits/responsys-rate-limits.yml`.
- Renaming a list, PET, table or folder in the Responsys UI changes its API address and
  breaks every hard-coded path.
