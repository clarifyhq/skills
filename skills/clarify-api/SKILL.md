---
name: clarify-api
description: Guidelines for working with the Clarify CRM API - authentication, conventions, errors, rate limits, and the full endpoint surface (records, schemas, custom objects, relationships, merges, attachments, lists, comments, meetings/transcripts, workflows, users)
author: clarify
version: 1.1.0
---

# Clarify API

> **Note:** The Clarify API is under active development (OpenAPI `version: 0.1`, Beta) and subject to breaking changes. Contact support@clarify.ai with any questions.
>
> This skill is generated against the authoritative OpenAPI spec at `https://api.clarify.ai/swagger-json` (last synced 2026-06-28). The hand-written docs site can lag the live contract, so when any detail here disagrees with the spec, **the spec wins** — `curl -o clarify-openapi.json https://api.clarify.ai/swagger-json`.

## Root URL

```
https://api.clarify.ai/v1/workspaces/{slug}/*
```

Every endpoint is workspace-scoped — there are no top-level paths. The `{slug}` is your workspace ID (the OpenAPI path parameter is named `workspace`). Find it in the URL when you log in, or by clicking your avatar in the app. All paths below are relative to `https://api.clarify.ai/v1/workspaces/{slug}`.

> Omitting `/v1` returns 404.

## Authentication

Two methods: **API key** (direct integrations) and **OAuth 2.0** (partner apps).

### API Key

Generate from **Settings → API Keys → Create API key** (workspace admin only; the key is shown once). Keys have admin-level permissions scoped to a single workspace: read/write/delete records, redacted email metadata only (no bodies), same meeting-data visibility as other users.

Send it in the `Authorization` header with the **`api-key`** scheme — **not** `Bearer`:

```bash
curl --header "Authorization: api-key $APIKEY" \
  https://api.clarify.ai/v1/workspaces/$WORKSPACE_SLUG/schemas
```

> Using `Bearer` with an API key → `401`. This is the most common mistake. `Bearer` is for OAuth access tokens only (below).

### Partner OAuth 2.0

Supports Authorization Code, Authorization Code with PKCE, and Refresh Token grants. Contact support@clarify.ai to register an OAuth app.

**Endpoints:**
- Authorization: `https://auth1.clarify.ai/oauth2/authorize`
- Token / Refresh: `https://auth1.clarify.ai/oauth2/token`
- Scopes: `openid profile email offline_access`

```http
GET https://auth1.clarify.ai/oauth2/authorize?response_type=code&client_id={your_client_id}&scope=openid%20profile%20email%20offline_access&redirect_uri={your_redirect_url}
```

```http
POST https://auth1.clarify.ai/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
code={your-code}
redirect_uri={your_redirect_uri}
client_id={your_client_id}
client_secret={your_client_secret}
```

Then call the API with the access token:
```
Authorization: Bearer {access-token}
```

## API Conventions

The API follows the [JSON:API spec](https://jsonapi.org/). Records are wrapped as `{ "data": { "type", "id", "attributes": {...} } }`. **Comments are an exception** — they use a flat (non-wrapped) body.

### Pagination (offset-based)

Query params on list/`resources` endpoints:
- `page[limit]={number}` — page size (default **1000**)
- `page[offset]={number}` — starting offset (default **0**)

Response:
- `data` — array of items
- `meta` — `total_records`, `total_pages`, `offset`, `limit`
- `links` — `prev` / `next` (when available). Follow `links.next` until absent.

### Ordering

- `sortOrder[column]={field}` — field to order by
- `sortOrder[dir]=ASC|DESC`

### Filtering

Conditions are **AND only** (OR is not supported).

```
filter[{field}]={value}                  # basic
filter[{field}][{Operator}]={value}      # with operator
```

> **Operators are case-sensitive** (`Contains`, not `contains`). For text fields use `Contains`, not `Equals` (`Equals` on text → 400). With `curl`, pass `--globoff` so the `[ ]` aren't shell-expanded, and URL-encode spaces in operator names (`Greater%20than`).

| Field Type | Operators |
|------------|-----------|
| Text, Multiline Text | `Contains`, `Does not contain`, `Starts with`, `Ends with`, `Is`, `Is not` |
| Number, Currency | `Is`, `Greater than`, `Less than`, `Greater than or equal`, `Less than or equal` |
| Dates | `Is`, `Is after`, `Is before`, `Is on or after`, `Is on or before` |
| Single select | `Is`, `Is not`, `One of`, `Not one of` |
| Multi select | `Contains`, `Does not contain` |
| Checkbox / Boolean | `Is`, `Is not` |
| Location | `Contains`, `Does not contain` |
| Record IDs | `Is`, `Is not` |
| Collections (Email, Domains, Participants) | `Contains`, `Does not contain` |

Most field types also support `Is empty` / `Is not empty`. There is no OR and no full-text search across fields.

### Including relationships

`include={rel1,rel2}` (comma-separated) on `GET .../resources` and `GET .../records/{id}` expands related records inline (e.g. `include=companies,deals`).

### Field-value formats (gotchas)

- **Collection fields** (`email_addresses`, `domains`, phone numbers) take an object, not a bare array: `{ "items": ["jane@example.com"] }`. A plain array → 422. `{ "items": [] }` clears the field.
- **Rich-text fields** (`task.description`, comment `message`) are a block-array JSON (BlockNote format), e.g. `[ { "type": "paragraph", "content": [ { "type": "text", "text": "..." } ] } ]`.
- **Enum / single-select values are case-sensitive** (`"Active"` ≠ `"active"`).
- **Dates** are `YYYY-MM-DD`; declare new date fields as `"type": "date"` (not `date-time`, which becomes read-only).

## Errors

Envelope:
```json
{
  "errors": [
    { "status": 422, "title": "Validation Error", "detail": "Duplicate record found for email: jane@example.com" }
  ]
}
```

| Code | Common cause |
|------|--------------|
| 400 | Duplicate on unique field (use PATCH/upsert); `Equals` used on a text field |
| 401 | Wrong auth scheme (`Bearer` instead of `api-key`) or invalid key |
| 404 | Unknown record id; missing `/v1` in the URL; bulk PATCH with any missing id |
| 422 | `id` sent in a POST create; enum casing mismatch; invalid email; bare array for a collection field; `date-time` instead of `date` |
| 429 | Rate limited — back off exponentially |
| 500 | Single-record PATCH on a **custom** object (use bulk PATCH instead) |

## Rate limits

No strict per-second enforcement, but Clarify recommends: **≤25 records per bulk batch**, ~**1s between batches**, ~**100 requests/minute** for reads. Exceeding → 429; use exponential backoff.

---

## Endpoints

Object types (`{object}` in paths): `person`, `company`, `deal`, `meeting`, `task`, plus any custom object (slug prefixed `c_`). Get the live list and field schemas from `GET /schemas`.

### Schemas & fields

```http
GET    /schemas                                   # all object schemas (core/* + entities/*)
PATCH  /schemas                                   # update enum field values (body: { data })
POST   /schemas/{entity}/properties               # add custom fields (body: { data: { type:"schema-properties", attributes } })
POST   /schemas/{entity}/relationships            # define a relationship field
DELETE /schemas/{entity}/relationships/{fieldName}# remove a relationship definition + its data
PATCH  /schemas/{entity}/fields-order             # reorder fields (body: { data })
PATCH  /schemas/{entity}/fields-visibility        # show/hide fields (body: { data })
```

`{entity}` is the full schema key, e.g. `core/person` or `entities/c_sales_order`. The relationship-definition body is large (`xClarifyRelationship`, `kind` one of `many-to-one`/`one-to-many`/`many-to-many`, optional AI-field config) — see the Swagger spec for the full shape. Removing enum values is destructive (nulls affected records).

### Custom objects

```http
POST   /schemas/objects                # create a custom object type
PUT    /schemas/objects/{object}       # replace a custom object's schema (async → returns async-task)
DELETE /schemas/objects/{object}       # delete a custom object + all its records (async; may be disabled per-workspace)
```

Create body (required: `name`, `plural`):
```json
{
  "name": "Sales Order",
  "plural": "Sales Orders",
  "description": "Customer purchase orders and invoices"
}
```
`name` (1–32 chars) is normalized to a `c_` identifier (`Sales Order` → `c_sales_order`). Custom objects reuse every record endpoint below with their `c_` name. **Single-record PATCH is not supported on custom objects → 500; use the bulk `PATCH /objects/{object}/records` instead.**

### Records — read

```http
GET  /objects/{object}/resources           # list records (pagination, sortOrder, filter, include)
GET  /objects/{object}/resources/{id}       # get one resource
POST /objects/{object}/resources            # create (JSON:API-style alias of POST .../records; body: { data })
GET  /objects/{object}/records/{id}         # get one record (supports ?include=...)
```

> The record **collection** is read via `.../resources`, not `.../records` — there is no `GET .../records`. `.../records` is create/update/delete only. `POST .../resources` and `POST .../records` are equivalent create paths; the examples below use `.../records`.

### Records — create / update / delete

```http
POST   /objects/{object}/records          # create one (upserts on unique-field match)
PATCH  /objects/{object}/records/{id}      # update one (only included fields change)
DELETE /objects/{object}/records/{id}      # delete one (204, no body)
POST   /objects/{object}/records/bulk      # bulk create/upsert (body: { data: [...] })
PATCH  /objects/{object}/records           # bulk update (body: { data: [...] }, each needs id)
DELETE /objects/{object}/records           # bulk delete (body: { items: [id, ...] })
```

Create — do **not** send `id` (→ 422):
```json
{
  "data": {
    "type": "{object}",
    "attributes": { "{field-name}": "{field-value}" }
  }
}
```

Update one:
```json
{
  "data": {
    "id": "{id}",
    "type": "{object}",
    "attributes": { "{field-name}": "{field-value}" }
  }
}
```

Bulk update — every item needs `id`; all ids are pre-validated (404 if any is missing); omitted attributes are preserved. Optional `meta` controls per-field merge strategy (`collection`: `append`/`remove`/`replace`):
```json
{
  "data": [
    { "type": "{object}", "id": "{id}", "attributes": { "{field}": "{value}" } }
  ]
}
```

Bulk delete (`items` is an array of id strings, min 1):
```json
{ "items": ["rec_1", "rec_2"] }
```

**Unique fields for upsert** (create/bulk match on these):

| Object | Unique Field |
|--------|--------------|
| Person | `email_addresses` (any one) |
| Company | `domains` (any one) |
| Deal | `name` |
| Meeting / Task | none (always creates) |
| Custom | as configured (check `GET /schemas`) |

### Record relationships

```http
GET    /objects/{object}/records/{id}/relationships/{relationship}   # list related records
PATCH  /objects/{object}/records/{id}/relationships/{relationship}   # replace the full related set
DELETE /objects/{object}/records/{id}/relationships/{relationship}   # remove specific related records
```

PATCH **replaces** the entire set (not additive); supports multiple records. Body for PATCH/DELETE:
```json
{
  "data": [
    { "id": "{related-id}", "type": "{object}" }
  ]
}
```
Known relationships: person↔`companies`, deal↔`companies`, deal↔`people`, meeting↔`people`. Relationships can also be set at creation via FK fields (e.g. `company_id`).

### Record merges & attachments

```http
POST   /objects/{object}/records/{record}/merges                      # merge source records into one
GET    /objects/{object}/records/{id}/attachments                     # list attachments
PUT    /objects/{object}/records/{id}/attachments                     # get a signed upload URL (body: { name })
POST   /objects/{object}/records/{id}/attachments                     # register an uploaded file (body: { name, key, attachmentId })
GET    /objects/{object}/records/{id}/attachments/{attachmentId}      # get one attachment
DELETE /objects/{object}/records/{id}/attachments/{attachmentId}      # delete an attachment
GET    /objects/{object}/records/{record}/activities                  # record activity timeline
```

Merge body — `sources` is the list of record ids to fold in:
```json
{ "data": { "type": "{object}", "attributes": { "sources": ["rec_a", "rec_b"] } } }
```

### Lists

```http
GET    /lists                                          # all lists in the workspace
GET    /objects/{object}/lists                         # lists scoped to an object type
POST   /objects/{object}/lists                         # create a list
GET    /objects/{object}/lists/{list}                  # get one list
PATCH  /objects/{object}/lists/{list}                  # update a list
DELETE /objects/{object}/lists/{list}                  # delete a list
POST   /objects/{object}/lists/{list}/publish          # publish a list
POST   /objects/{object}/lists/{list}/unpublish        # unpublish a list
GET    /objects/{object}/lists/{list}/resources        # records in a list
POST   /objects/{object}/lists/{list}/rows/csv         # CSV export of list rows
```

Create body (required: `title`, `description`, `layout`, `permission`, `type`). `type` is `static` or `dynamic`; dynamic lists carry a `query` (`{ sql, version: 6 }`).

### Comments

Comments are **flat** (not JSON:API-wrapped):

```http
POST   /comments        # create
GET    /comments/{id}    # get one (by UUID)
PATCH  /comments/{id}    # update (only message editable)
DELETE /comments/{id}    # delete
```

Body (required: `message`, `owner_id`, `entity`):
```json
{
  "message": [
    { "type": "paragraph", "content": [ { "type": "text", "text": "Follow up in two weeks." } ] }
  ],
  "owner_id": "872aced7-8f28-4bc8-9c2b-2602cb943c0d",
  "entity": "person"
}
```
`owner_id` is the target record's id; `entity` is its object type (`person`/`company`/`deal`/`meeting`/`c_*`).

### Meetings, recordings & transcripts

```http
POST   /meetings/{meetingId}/recording                       # start recording (no body)
DELETE /meetings/{meetingId}/recording                       # stop recording (204)
POST   /meetings/{meetingId}/recordings/{id}/artifacts       # signed URLs for a recording
POST   /meetings/{meetingId}/transcript                      # upload a transcript (body: { transcript: [...] })
```

Artifacts response (note: **camelCase**, short-lived signed URLs):
```json
{ "data": { "videoUrl": "{signed-video-url}", "transcriptionUrl": "{signed-transcription-url}" } }
```

Transcript upload — `transcript` is an array of speaker segments; each segment has `words[]` (`text`, `start_timestamp`, `end_timestamp`, `language`, `confidence`), `speaker`, `speaker_id`, `language`:
```json
{
  "transcript": [
    {
      "speaker": "Jane Doe",
      "speaker_id": 0,
      "language": "en",
      "words": [
        { "text": "Hello", "start_timestamp": 0.0, "end_timestamp": 0.4, "language": "en", "confidence": 0.98 }
      ]
    }
  ]
}
```
The meeting `notes` field is not writable via the API.

### Tasks

Tasks use the record endpoints with object type `task` (always creates — no upsert). Fields: `title` (required), `description` (BlockNote array), `status` (`To Do`/`In Progress`/`Done`/`Canceled`), `priority` (`Urgent`/`High`/`Medium`/`Low`), `due_date` (`YYYY-MM-DD`), `assignee_id` (user id). FK fields: `person_id`, `company_id`, `deal_id`, `meeting_id`.

### Workflows

```http
GET    /workflows          # list
POST   /workflows          # create (body: { data })
GET    /workflows/{id}      # get one
PATCH  /workflows/{id}      # update (body: { data })
DELETE /workflows/{id}      # delete
```

### Users

```http
GET /users           # list workspace users
GET /users/{userId}   # get one user
```

User ids referenced elsewhere (`assignee_id`, `_created_by`, `_updated_by`) are opaque (WorkOS / OAuth ids, or `system`).

## Not in the API

Email content/bodies, writing meeting `notes`, sending email, and outbound webhooks are not available. For near-realtime sync, poll with `filter[_updated_at][Is after]=...` or use Segment Reverse ETL.

## Full reference

Interactive explorer: `https://api.clarify.ai/swagger` · raw spec: `https://api.clarify.ai/swagger-json` (OpenAPI 3.1; 55 endpoints). Generate a typed client: `npx openapi-typescript https://api.clarify.ai/swagger-json -o clarify-api.d.ts`.
