---
name: clarify-api
description: Guidelines for working with the Clarify CRM API - authentication (API keys + MCP), conventions, filtering (incl. shorthand operators and relationship filters), errors, rate limits, and the full endpoint surface (records, schemas, custom objects, relationships, merges, attachments, access control, lists, comments, meetings/transcripts, workflows, campaigns, users, settings, layouts)
author: clarify
version: 1.1.0
---

# Clarify API

> **Note:** The Clarify API is under active development (OpenAPI `version: 0.1`, Beta) and subject to breaking changes. Contact support@clarify.ai with any questions.
>
> This skill is generated against the authoritative OpenAPI spec at `https://api.clarify.ai/swagger-json` (last synced 2026-08-20) and cross-checked against the docs at `https://developer.clarify.ai/docs`. The hand-written docs site can lag the live contract, so when any detail here disagrees with the spec, **the spec wins** — `curl -o clarify-openapi.json https://api.clarify.ai/swagger-json`.

## Root URL

```
https://api.clarify.ai/v1/workspaces/{slug}/*
```

Every endpoint is workspace-scoped — there are no top-level paths. The `{slug}` is your workspace ID (the OpenAPI path parameter is named `workspace`). Find it in the URL when you log in, or by clicking your avatar in the app. All paths below are relative to `https://api.clarify.ai/v1/workspaces/{slug}`.

> Omitting `/v1` returns 404.

## Authentication

Three ways to connect: **API key** (direct integrations), **MCP server** (agents), and **OAuth 2.0** (partner apps).

### API Keys

Generate from **Settings → API Keys**. Keys come in two types:

| Type | Who creates | Scope | Use for |
|------|-------------|-------|---------|
| **Personal** | Any workspace member | Everything the creator can access | MCP, direct API, personal scripts |
| **Workspace** | Workspace admins only | Public data within the workspace | Shared backend services |

Keys have read/write/delete record access, redacted email metadata only (no bodies), and the same meeting-data visibility as other users. Treat keys like passwords: store in an environment variable, never commit them, rotate periodically.

Send it in the `Authorization` header with the **`api-key`** scheme — **not** `Bearer`:

```bash
curl --header "Authorization: api-key $APIKEY" \
  https://api.clarify.ai/v1/workspaces/$WORKSPACE_SLUG/schemas
```

> Using `Bearer` with an API key → `401`. This is the most common mistake. `Bearer` is for OAuth access tokens only (below).

### MCP Server

Clarify hosts an MCP server at `https://api.clarify.ai/mcp` — compatible with Claude Code, Claude Desktop, Cursor, Windsurf, or custom agents. Authenticate with a **Personal** API key plus workspace slug:

```
Authorization: api-key YOUR_API_KEY
X-Clarify-Workspace: your-workspace
```

> **Only Personal keys are accepted; workspace keys are rejected** by the MCP server.

### Partner OAuth 2.0

Supports Authorization Code, Authorization Code with PKCE, and Refresh Token grants. Contact support@clarify.ai to register an OAuth app (describe the integration, public vs private, redirect URIs, optional logo).

- **Private apps** (server, can keep a secret): Authorization Code flow with Client Secret.
- **Public apps** (native/SPA, no secret): Authorization Code with PKCE (`code_challenge_method=SHA-256`), `client_id` in the token request body.

**Endpoints:**
- Authorization: `https://auth1.clarify.ai/oauth2/authorize`
- Token / Refresh: `https://auth1.clarify.ai/oauth2/token`
- Scopes: `openid profile email offline_access` (`offline_access` is required to receive a refresh token)

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
- `page[limit]={number}` — page size (spec default **500**; the live API rejects values over **500**, so use `500` for bulk reads)
- `page[offset]={number}` — starting offset (default **0**)

Response:
- `data` — array of items
- `meta` — `total_records`, `total_pages`, `offset`, `limit`
- `links` — `prev` / `next` (both always present; `null` on the first/last page). Follow `links.next` until absent.

### Ordering

- `sortOrder[column]={field}` — field to order by
- `sortOrder[dir]=ASC|DESC`

### Filtering

Conditions are **AND only** (OR is not supported).

```
filter[{field}]={value}                  # basic
filter[{field}][{Operator}]={value}      # with operator
```

> **Operators are case-sensitive** (`Contains`, not `contains`). There is **no `Equals` operator** — use `Is` for exact match, `Contains` for partial; unknown operators → 400. With `curl`, pass `--globoff` so the `[ ]` aren't shell-expanded, and URL-encode spaces in operator names (`Greater%20than`).

| Field Type | Operators |
|------------|-----------|
| Text, Multiline Text | `Is`, `Is not`, `Contains`, `Does not contain`, `Starts with`, `Ends with`, `Is empty`, `Is not empty` |
| Number, Currency | `Is`, `Is not`, `Greater than`, `Greater than or equal`, `Less than`, `Less than or equal`, `Is empty`, `Is not empty` |
| Dates | `Is`, `Is not`, `Is before`, `Is on or before`, `Is after`, `Is on or after`, `Is empty`, `Is not empty` |
| Single select | `Is`, `Is not`, `One of`, `Not one of`, `Is empty`, `Is not empty` |
| Multi select | `Contains`, `Does not contain`, `Is empty`, `Is not empty` |
| Checkbox / Boolean | `Is`, `Is not`, `Is set`, `Is not set` |
| Location | `Contains`, `Does not contain` |
| Record IDs | `Is`, `Is not`, `Is empty`, `Is not empty` |
| Collections (Email, Domains, Participants) | `Contains`, `Does not contain`, `Is empty`, `Is not empty` |

Collection and multi-select fields have **no exact-match operator** — one must be named explicitly. Most field types also support `Is empty` / `Is not empty`. There is no OR and no full-text search across fields.

**Shorthand operators** (equivalent to the named forms above):

| Shorthand | Equivalent | Field types |
|-----------|-----------|-------------|
| `filter[amount]=>50000` | `Greater than` | Number, date |
| `filter[amount]=>=50000` | `Greater than or equal` | Number, date |
| `filter[amount]=<50000` | `Less than` | Number, date |
| `filter[amount]=<=50000` | `Less than or equal` | Number, date |
| `filter[stage]=!=Lost` | `Is not` | Number, date, text, select, boolean |
| `filter[name]=*Smith*` | `Contains` | Text |
| `filter[name]=Smith*` | `Starts with` | Text |
| `filter[name]=*Smith` | `Ends with` | Text |
| `filter[stage]=Won,Negotiation` | `One of` | Select, multi select |
| `filter[stage]=null` | `Is empty` | Most field types |
| `filter[stage]=!null` | `Is not empty` | Most field types |

**Range queries** — combine two operators on the same field; both conditions are ANDed:
```
filter[amount][Greater than]=10000&filter[amount][Less than or equal]=50000
```

**Filtering on related records** — use a dotted path starting with the queried object, then the relationship field and target field (traverses up to **three levels deep**; no `include` needed to filter):
```bash
curl --globoff -H "Authorization: api-key $APIKEY" \
  "https://api.clarify.ai/v1/workspaces/$WS/objects/person/resources?filter[person.company_id.name]=Acme"
```

### Including relationships

`include={rel1,rel2}` (comma-separated) on `GET .../resources` and `GET .../records/{id}` expands related records inline (e.g. `include=companies,deals`). `include` only affects the response — it is not needed to filter on a relationship.

### Field-value formats (gotchas)

- **Collection fields** (`email_addresses`, `domains`, phone numbers) take an object, not a bare array: `{ "items": ["jane@example.com"] }`. A plain array → 422. `{ "items": [] }` clears the field.
- **Rich-text fields** use BlockNote JSON. A comment's `message` is the block array directly; a **task's `description` wraps the same block array in an object under a `text` key** — `{ "text": [ ...blocks ] }`. Passing a plain string → 422.
  - Block types: `paragraph`, `heading` (with `props.level`: 1–6), `bulletListItem`, `numberedListItem`, `checkListItem`, `codeBlock`.
  - Text runs: `{ "type": "text", "text": "...", "styles": { "bold": true, "italic": true, "underline": true, "strike": true, "code": true, "textColor": "red", "backgroundColor": "yellow" } }` (styles are booleans plus color-name strings).
  - Simplest valid payload: one `paragraph` block with a single text run.
- **Enum / single-select values are case-sensitive** (`"Active"` ≠ `"active"`).
- **Dates** are `YYYY-MM-DD`; declare new date fields as `"type": "date"` (not `date-time`, which becomes read-only).

## Errors

Envelope (JSON:API; `status` is a **string**; `title` present on validation errors, omitted otherwise; `source.pointer` is a JSON Pointer to the offending field on validation errors):
```json
{
  "errors": [
    { "status": "422", "title": "Invalid input", "detail": "/data/attributes/email: invalid email", "source": { "pointer": "/data/attributes/email" } }
  ]
}
```

| Code | Common cause |
|------|--------------|
| 400 | Duplicate on unique field (use `match_on`/PATCH); `Equals` used on a text field; numeric value out of range (must fit `DECIMAL(16, 4)`); malformed JSON body; missing workspace slug |
| 401 | Wrong auth scheme (`Bearer` instead of `api-key`) or invalid key |
| 404 | Unknown record id; missing `/v1` in the URL; bulk PATCH with any missing id; unknown workspace slug; unknown `{object}` path segment |
| 409 | Schema being modified by another request (`"Schema is being modified by another request. Please retry."`) — retry after a short delay |
| 422 | `id` sent in a POST create; `_`-prefixed system field in `attributes`; enum casing mismatch; invalid email; bare array for a collection field; `date-time` instead of `date`; to-many relationship set as an attribute |
| 429 | Rate limited — honor `Retry-After`, then back off exponentially |
| 500 | Unexpected server error (retry with backoff; contact support if it persists). Single-record PATCH on a **custom** object also → 500 (use bulk PATCH instead) |

## Rate limits

**3,000 requests per minute, per workspace, per endpoint** (rolling 60-second window; each endpoint has its own budget, so heavy traffic on one doesn't throttle others). Exceeding → 429 until the window resets.

Rate-limit headers on every response:
- `X-RateLimit-Limit` — max requests in the window (`3000`)
- `X-RateLimit-Remaining` — requests remaining
- `X-RateLimit-Reset` — seconds until the window resets
- `Retry-After` — seconds to wait (added on 429 responses)

Backoff: wait the `Retry-After` seconds; if the header is missing, fall back to exponential backoff (`2 ** attempt`). For high-volume writes, use the bulk endpoints (one request with many records counts as a single request) and add a short delay between batches.

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
GET    /schemas/{entity}/activities               # schema change activity
```

For custom fields on built-in CRM objects, the live API expects the object slug in `{entity}`, e.g. `person` or `company`; `core/person`, `entities/person`, and URL-encoded `core%2Fperson` return 404. The created fields appear under `entities/person` or `entities/company` in `GET /schemas`.

The relationship-definition body is large (`xClarifyRelationship`, `kind` one of `many-to-one`/`one-to-many`/`many-to-many`, optional AI-field config) — see the Swagger spec for the full shape. **Editing enum values is partly destructive**: *adding* values is safe; *removing* values nulls affected records (single-select clears to `null`, multi-select drops the option) and drops them from lists filtering on the value — check which records use a value before removing it.

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
`name` (1–32 chars) is normalized to a `c_` identifier (`Sales Order` → `c_sales_order`). If the normalized name already exists, creation is rejected with a **422**. Custom objects reuse every record endpoint below with their `c_` name. **Single-record PATCH is not supported on custom objects → 500; use the bulk `PATCH /objects/{object}/records` instead.**

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

**Upsert with `match_on`:** Creating is a plain insert by default — a value that collides with an existing record's unique field is rejected with a **400**. To update the existing record instead, pass `match_on` set to the object's unique field (top-level, next to `data`). Without it, a re-run fails on every record already in the workspace.

| Object | Unique Field (`match_on`) |
|--------|--------------|
| Person | `email_addresses` (any one) |
| Company | `domains` (any one) |
| Deal | `name` |
| Meeting / Task | none (always creates) |
| Custom | as configured (check `GET /schemas`) |

**Import order** (so foreign keys resolve on the first pass): 1) Companies (no dependencies) → 2) People (can reference `company_id`) → 3) Deals (can reference `company_id`) → 4) Associations (link people to deals via relationships) → 5) Transcripts (link to meetings and people).

**Batch sizing** (bulk endpoints have no fixed cap — the practical ceiling is request body size; batches are **atomic**, one bad record fails the whole batch): clean validated data → 100; first import with unknowns → 25; retrying failures → 1 (isolate the bad record).

**To-one vs to-many relationships:** a to-one relationship (e.g. `company_id`) is a regular field on the record and settable like any other attribute. To-many relationships (e.g. people↔deals) **cannot** be set as attributes on create (→ 422); they're managed through the relationships endpoint.

### Record relationships

```http
GET    /objects/{object}/records/{id}/relationships/{relationship}   # list related records
PATCH  /objects/{object}/records/{id}/relationships/{relationship}   # replace or append (see below)
DELETE /objects/{object}/records/{id}/relationships/{relationship}   # remove specific related records
```

PATCH behavior depends on the relationship kind:
- **Many-to-many** (e.g. people on a deal): **replaces the entire set** — any related records not in the body are unlinked.
- **One-to-many** (e.g. a company's people): **additive** — links the records you send, leaves the rest attached; linking a record that already belongs to another parent reassigns it.

To remove specific records in either case, use `DELETE` on the same path. Body for PATCH/DELETE:
```json
{
  "data": [
    { "id": "{related-id}", "type": "{object}" }
  ]
}
```
Known relationships: person↔`companies`, deal↔`companies`, deal↔`people`, meeting↔`people`. Relationships can also be set at creation via FK fields (e.g. `company_id`).

### Import hygiene

For API imports that need a saved list later, do **not** put operational markers like campaign, source, batch, or status in `description`. `description` is visible in record headers and pollutes the CRM UI.

Preferred pattern:
1. Create a dedicated custom field, e.g. `import_batch`.
2. Bulk upsert records with that field set to a stable batch value.
3. Build dynamic lists from the custom field.
4. Leave `description` empty unless it is useful human-facing CRM context.

When importing enriched contact data, preserve the source's name on person records even when the selected email is role-based or company-generic — don't replace a real source name with a placeholder like `{Company} team`; let `job_title` and the company relationship carry the business context.

When parsing enriched emails from CSVs, split multi-value cells on `|`, `,`, and `;`, then validate the selected email/domain before sending bulk writes. A single invalid hostname makes the whole bulk batch fail atomically.

CSV export cleanup: strip `http://`, `https://`, and `www.` from domains; split comma-separated emails in one cell (an email can belong to only one person); import contacts from catch-all companies ("Website Leads", "Unknown Company") without a company link rather than recreating the catch-all.

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

Merge body — `sources` is the list of record ids to fold in (field values and relationships combine onto the target; sources are deleted; cannot be undone):
```json
{ "data": { "type": "{object}", "attributes": { "sources": ["rec_a", "rec_b"] } } }
```

### Access control (record sharing)

Shareable objects are `list`, `meeting`, and `message`. A grant shares a record with a single workspace member or the whole workspace at a given access level.

```http
GET    /objects/{object}/records/{id}/access              # list access grants on a record
POST   /objects/{object}/records/{id}/access              # create a grant (share with a user or the whole workspace)
POST   /objects/{object}/records/{id}/access/bulk         # create grants in bulk (up to 1000 per call)
PATCH  /objects/{object}/records/{id}/access/{grantId}    # update a grant
DELETE /objects/{object}/records/{id}/access/{grantId}    # revoke a grant
GET    /objects/{object}/access                           # list entity-wide access delegations
POST   /objects/{object}/access                           # grant entity-wide delegation (same access you have to every record of the object type; requires paid plan + user-backed API key)
DELETE /objects/{object}/access/{id}                      # revoke an entity-wide delegation
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

Create body (required: `title`, `description`, `layout`, `type`). `type` is `static` or `dynamic`; dynamic lists carry a `query` (`{ sql, version: 6 }`). Optional: `emoji` (shown next to the list), `options` (view-level display options), `rank` (sort position), `state` (`draft` or `published`, default `published`), and `silent` (query param — suppresses in-app/Slack notifications for the mutation).

**Dynamic list SQL conventions:**
- Columns must be fully qualified with the object type. Record-link columns use the alias pattern `"entity:__object__"` (e.g. `"deal".name AS "deal:__object__"`); relationship columns use `"entity$field:__object__"` and require a LEFT JOIN (without the JOIN they render as raw UUIDs).
- Multi-select fields (including built-in `labels`) store JSONB `{ "items": [...] }`; filter with the array-overlap operator `?|` — `WHERE ("deal".labels->'items') ?| ARRAY['hot-lead']`. Use `?|`, **not** `@>`: `@>` works in the DB but the list UI renders it as uneditable raw SQL instead of the native filter widget.
- Filter patterns: string equals `UPPER("deal".field) = UPPER('value')`; contains `"deal".field ILIKE '%value%'`; select `"deal".stage = 'Won'`; one-of `"deal".stage IN ('Won','Lost')`; number `"deal".amount > 1000`; date `"deal".close_date >= '2026-01-01'`; boolean `"deal".is_active = true`.
- When updating `query`, the **entire query object is replaced** — GET the list first, merge your changes into the existing SQL, then PATCH with the complete query.
- `version: 6` is the current query format; omitting it runs the SQL through query migrations on save, which can rewrite it.

**UI-visible list gotcha:** Raw REST-created dynamic lists can return 200/201 by ID and export rows, but still fail or not appear correctly in the Clarify web UI. Raw `PATCH /objects/{object}/lists/{list}` may also update metadata like `_updated_at` while leaving `query.sql` unchanged. For user-facing lists or query changes, create/update the list in the UI until the API returns a UI-compatible payload.

List titles should not include record counts like `(40)`. Clarify shows counts itself; counts in titles go stale.

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
`owner_id` is the target record's id; `entity` is its object type (`person`/`company`/`deal`/`meeting`/`c_*`). Comments surface in the record's activity stream; there is no separate list-comments endpoint.

### Meetings, recordings & transcripts

```http
POST   /meetings/{meetingId}/recording                       # start recording (no body)
DELETE /meetings/{meetingId}/recording                       # stop recording (204)
POST   /meetings/{meetingId}/recordings/{id}/artifacts       # signed URLs for a recording
POST   /meetings/{meetingId}/recordings/media                # start a media upload → presigned S3 POST policy (external audio/video)
POST   /meetings/{meetingId}/recordings/transcript           # upload a transcript for an externally-captured recording
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

Tasks use the record endpoints with object type `task` (always creates — no upsert). Fields: `title` (required), `description` (BlockNote array wrapped as `{ "text": [...] }`), `status` (`To Do`/`In Progress`/`Done`/`Canceled`), `priority` (`Urgent`/`High`/`Medium`/`Low`), `due_date` (`YYYY-MM-DD`), `assignee_id` (user id). FK fields: `person_id`, `company_id`, `deal_id`, `meeting_id`.

### Workflows

```http
GET    /workflows          # list
POST   /workflows          # create (body: { data })
GET    /workflows/{id}      # get one
PATCH  /workflows/{id}      # update (body: { data })
DELETE /workflows/{id}      # delete
```

Workflows are available on request (contact support@clarify.ai to enable). A workflow runs steps when a trigger fires (record created/updated/deleted, list membership change, inbound webhook, schedule). **Execute code** steps run Node.js with a 60-second budget and only `lodash` (injected as `_`) plus the pre-authenticated `clarify` SDK — importing other npm packages fails the step.

```javascript
export async function handler(input, { clarify, _, workflowContext }) {
  // input: the step's configured input
  // clarify: pre-authenticated SDK client (don't read an API key or use raw fetch)
  // workflowContext.trigger.event.object.data — the triggering record (fields read directly, e.g. .amount)
  // workflowContext.trigger.event.object._id — the triggering record's ID
  // workflowContext.blocks['my-step'].result — earlier step outputs
  const response = await clarify.resources.list({
    type: "company",
    filter: { industry: "technology" },
    page: { limit: 50, offset: 0 },
  });
  const { data } = await clarify.records.get("deal", dealId);
  await clarify.records.update("deal", dealId, { stage: "negotiation" });
  await clarify.records.createBulk("person", [...]);
  return result; // must be JSON-serializable; throw to fail the run
}
```

SDK notes: trigger data exposes fields directly (`deal.amount`), but SDK responses nest them under `.attributes`. Entity slugs are always lowercase. Self-triggering is prevented (Clarify skips the run when the change was made by that same workflow); cross-workflow cycles are not.

### Campaigns

```http
GET /campaigns/{campaignId}/recipients   # people enrolled in a campaign, with per-recipient engagement
```

Each recipient includes whether they opened, clicked, replied, or unsubscribed, and when each last happened. Filter by engagement with `filter[engagement...]` (offset-paginated JSON:API collection).

### Users

```http
GET /users           # list workspace users
GET /users/{userId}   # get one user
```

User ids referenced elsewhere (`assignee_id`, `_created_by`, `_updated_by`) are opaque actor identifiers, not UUIDs. The value depends on who made the change: a workspace user's ID (format depends on the auth provider, e.g. `google-oauth2|106236165388989412707`), or a system actor (`system`, `agent`, or `rep`). Treat these as opaque; resolve user ids to names/emails via the Users endpoint — system-actor values are not workspace users and won't resolve there.

### Settings

```http
GET    /settings          # list workspace settings (defaults applied for unset keys)
POST   /settings          # write a setting by key (read-only settings rejected; some require admin)
DELETE /settings          # reset a setting to its default
GET    /settings/{key}    # get one setting
```

### Layouts

```http
GET    /layouts/{id}      # get a layout (defines UI structure: navigation, record page regions)
PATCH  /layouts/{id}      # replace the layout's `tree`
POST   /layouts/{id}/reset
```

## Webhooks

Webhooks are **not REST endpoints** — they're configured inside workflows and AI Agents, and connect Clarify to your stack in both directions:

- **Inbound** — an automation gets its own URL; a `POST` to it runs the automation (no `Authorization` header — the URL itself is the credential, treat it like a secret). Clarify responds `200 OK` on accept, then runs asynchronously. Body and headers are available as `{{trigger.state.event.data.body.*}}` / `{{trigger.state.event.data.headers.*}}` in workflows, or handed to the agent as context in AI Agents.
  ```
  https://api.clarify.ai/v1/webhooks/workflows/{workspace}/{workflow_id}
  https://api.clarify.ai/v1/webhooks/agents/{workspace}/{agent_id}
  ```
- **Outbound** — a workflow action (or agent code) sends an HTTP request to a URL you choose, with configurable method, headers, and body (all supporting variable interpolation). For dynamic payloads or branching on the response, use an Execute code step and `fetch` the URL directly.

## Not in the API

Email content/bodies, writing meeting `notes`, and sending email are not available. For near-realtime sync, poll with `filter[_updated_at][Is after]=...` or use Segment Reverse ETL (push warehouse changes to the bulk records API with `match_on` for upsert — hourly for fast-moving data, daily for reference data).

## Full reference

Interactive explorer: `https://api.clarify.ai/swagger` · raw spec: `https://api.clarify.ai/swagger-json` (OpenAPI 3.1; 75 operations across 48 paths). Generate a typed client: `npx openapi-typescript https://api.clarify.ai/swagger-json -o clarify-api.d.ts`. Docs site: `https://developer.clarify.ai/docs`.
