---
name: webscale-manage-edge-access-control
description: >-
  Block or allow traffic to a Webscale-hosted storefront by managing address sets, reading the web
  controls attached to an application, and inspecting address-set metrics. Use when asked to block an
  IP range or country, allow-list an office or partner, investigate a traffic spike, or audit which
  edge rules are live on an application.
api: Webscale APIs
base_url: https://api.webscale.com/v2
generated: '2026-09-04'
method: generated
source: openapi/webscale-networks-webscale-apis-openapi.json (Webscale APIs 2026.273)
operations:
  - GET /address-sets
  - POST /address-sets
  - GET /address-sets/{id}
  - PATCH /address-sets/{id}
  - DELETE /address-sets/{id}
  - GET /address-sets/{id}/addresses
  - GET /address-sets/{id}/metrics
  - GET /address-sets/metrics
  - GET /applications/{app_id}/webcontrols
  - GET /applications/{app_id}/webcontrols/{id}
  - GET /applications/{id}
  - PATCH /applications/{id}
---

# Manage edge access control on Webscale

> Operations are named by method + path because the Webscale OpenAPI declares no `operationId` on any
> of its 151 operations. Do not invent one.

## The model

An **application** is the service configuration for one web application. It carries two address-set
references — `whitelist_href` (allow) and `blacklist_href` (block), both typed `AddressSetHref`. An
**address set** "maintains a set of IP addresses usually used to block or allow access."

**Web controls** are the condition/action rules evaluated at the edge for an application, at
`/applications/{app_id}/webcontrols`. The published contract exposes them **read-only** — there is
`GET` on the collection and on an item, and **no POST, PATCH or DELETE**. To change a web control you
use the console at `https://control.webscale.com/`, not this API. Do not tell an operator you can
create a web control programmatically.

The web-control condition vocabulary in the contract includes: ASN is/is not, cookie set/not set,
cookie value matches, country of origin, IP in/not in set, IP allowlisted/not allowlisted, IP is/is
not a threat, random (probability), **rate_limit** (`client_id`, `duration`, `display_unit`,
`threshold`), referrer, request header, request method, response header, status code, URL, and user
agent.

## Steps

### Audit what is live on an application

```
GET /v2/applications/{id}
GET /v2/applications/{app_id}/webcontrols
```

Read `whitelist_href` and `blacklist_href` off the application, then fetch each set. There is **no
expand/include parameter** in this API, so each reference costs another round trip.

### Read the members of a set

```
GET /v2/address-sets/{id}
GET /v2/address-sets/{id}/addresses
```

Page with `start` and `limit` (default 100). Ordering is `order=` with `-`/`+` prefixes.

### Add or change entries

```
POST   /v2/address-sets        # create a new set
PATCH  /v2/address-sets/{id}   # update an existing set
```

`PATCH` is the update verb; `PUT` is not used anywhere in this API.

**No idempotency key exists.** A retried `POST /v2/address-sets` after a timeout can create a
duplicate set. Read back with a filter on the name before resending:

```
GET /v2/address-sets?filter=name = "<the name you sent>"
```

### Attach a set to an application

```
PATCH /v2/applications/{id}
```

Set `whitelist_href` or `blacklist_href` to the set's `href`.

> **This is a live traffic-control change with no undo.** The contract publishes no reversal
> operation and no reversal window. Capture the current value of the field you are about to overwrite
> **before** you patch, so you can restore it by hand:
>
> ```
> GET /v2/applications/{id}   # record blacklist_href / whitelist_href first
> ```
>
> Attaching an over-broad block list can take a storefront offline for real customers. Confirm the
> intended scope with a human before patching a production application.

### Measure the effect

```
GET /v2/address-sets/{id}/metrics
GET /v2/address-sets/metrics
```

Both take `from`, `to`, `resolution`, `summarize`, and the group family (`groupby`, `groupfilters`,
`grouplimit`, `grouporder`, `groupselect`). `format=csv` is available; when `format=json` the `limit`
must be 200 or less.

### Removing a set

```
DELETE /v2/address-sets/{id}
```

Detach it from every application that references it **first** — the contract declares no cascade
behaviour and no 4xx response telling you the set is in use. Deletion has no documented restore path.

## Errors and limits

- No 4xx/5xx responses are declared on any operation. Parse the `{type, description}` envelope
  defensively and treat any non-2xx as unexpected.
- **No API rate limit is published**, and no `RateLimit-*` / `Retry-After` header is declared. The
  "Basic/Advanced rate limiting" in Webscale's plans and the `rate_limit` web-control condition are a
  product feature that throttles traffic to the *customer's storefront* — they say nothing about how
  hard you may call this API. Back off on 5xx and pace conservatively.
