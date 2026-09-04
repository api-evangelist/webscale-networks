---
name: webscale-deploy-a-web-application
description: >-
  Stand up a new web application on the Webscale platform via the v2 control-plane API — inspect the
  environments and clusters available to the account, create the application, attach its allow/block
  address sets, and confirm it reads back. Use when an operator asks to onboard a storefront, add an
  application, or wire a domain to an existing Webscale cluster.
api: Webscale APIs
base_url: https://api.webscale.com/v2
generated: '2026-09-04'
method: generated
source: openapi/webscale-networks-webscale-apis-openapi.json (Webscale APIs 2026.273)
operations:
  - GET /environments
  - GET /clusters
  - POST /applications
  - GET /applications/{id}
  - POST /address-sets
  - PATCH /applications/{id}
  - GET /applications/{id}/metrics
---

# Deploy a web application on Webscale

> **Operation naming.** The Webscale OpenAPI declares **no `operationId` on any of its 151
> operations**, so every step below is identified by HTTP method + path, which is what the contract
> actually publishes. Do not invent operationId strings. If you need them, apply
> `overlays/webscale-networks-webscale-apis-overlay.yaml`, whose ids are API Evangelist's, not
> Webscale's.

## Before you start

- **Auth.** Every request carries `Authorization: Bearer <access-key-secret>`. Create the key in the
  user profile at `https://control.webscale.com/profile`, or mint one for a non-human principal with
  `POST /accounts/{id}/service-users`. The contract also accepts an `authorization` query parameter
  carrying the token — the header and the parameter are **mutually exclusive, exactly one**. Prefer
  the header; a token in a query string ends up in logs.
- **No idempotency.** There is no `Idempotency-Key` header and no client-supplied request key anywhere
  in this API. **A retried `POST` after a timeout can create a second application.** See the recovery
  step below rather than retrying blind.
- **No dry run.** Nothing here can be rehearsed. There is no validate-only or preview parameter.
- **No documented reversal.** No cancel, undo, rollback or restore operation exists in the contract,
  and no reversal window is published. Treat every `DELETE` as final unless you have confirmed
  otherwise with Webscale support.

## Steps

### 1. Find the environment and cluster to deploy into

```
GET /v2/environments?limit=100
GET /v2/clusters?limit=100
```

Both accept the shared collection parameters — `start` (default 1), `limit` (default 100), `order`,
and `filter`. Narrow with the filter expression language rather than paging everything:

```
GET /v2/clusters?filter=environment = "<environment-href>"
```

Filter grammar is published in full in the spec's `info.description`: relational
`= != ~ !~ < > <= >=`, `contains` / `not contains`, list `in` / `not in`, `is null` / `is not null`,
combined with `and` / `or` and parentheses.

Record the `href` of the cluster you choose. Webscale's model is href-addressed: **every addressable
resource carries an `href` attribute that is its own URI**, and that is the value other objects
reference.

### 2. Create the address sets first, if you need them

An application references its allow-list and block-list by href, so they must exist before you attach
them. Creating them first also means step 3 is a single write instead of a write plus a patch.

```
POST /v2/address-sets
```

### 3. Create the application

```
POST /v2/applications
```

Set `cluster` to the cluster href from step 1, and `whitelist_href` / `blacklist_href` to the address
sets from step 2 if you created them.

**This is the write that can duplicate.** If the call times out or the connection drops, do **not**
resend it. Go to the recovery step.

### 4. Confirm the read-back

```
GET /v2/applications/{id}
```

The response is the authoritative state. Check `current_outage` before reporting success, and note
that `cdn_state` is marked `deprecated: true` in the contract — do not build on it.

### 5. Adjust configuration

```
PATCH /v2/applications/{id}
```

`PATCH` is the only update verb in this API; `PUT` is not used on any path. Send only the fields you
are changing.

### 6. Verify it is serving

```
GET /v2/applications/{id}/metrics
```

The metrics surface takes `from`, `to`, `resolution`, `summarize`, and the group family
(`groupby`, `groupfilters`, `grouplimit`, `grouporder`, `groupselect`).

## Recovery from an ambiguous write

Because there is no idempotency key, a timed-out `POST /v2/applications` leaves you unable to tell
whether it landed. **Read back before you retry:**

```
GET /v2/applications?filter=name = "<the name you sent>"
```

- Zero results → the write did not land. Safe to resend.
- One result → the write landed. Do not resend.
- More than one → a previous retry already duplicated. Delete the extras with
  `DELETE /v2/applications/{id}`, and confirm you are deleting the right one by its `href`.

## Errors

The contract declares **no 4xx or 5xx response on any operation**, and its `Error` schema
(`{type, description}`) is defined but referenced by nothing. So:

- Do not branch on documented error types — there are none published.
- Do parse `{type, description}` defensively; that is the envelope shape the schema describes.
- Do not assume `429` means anything specific: no rate limit is published for this API, and no
  `RateLimit-*` or `Retry-After` header is declared. Back off exponentially on any 5xx.
- There is no request-id or correlation header, so there is no provider-issued handle to quote to
  support. Log your own request time, method, path and response body.
