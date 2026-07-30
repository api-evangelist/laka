---
name: laka-reporting-sync
description: Incrementally sync Laka policies, claims and invoices into your own system using the
  Platform reporting endpoints, their page-number pagination and updatedFrom/updatedTo filters —
  the substitute for webhooks Laka does not publish.
api: Laka Platform API
generated: '2026-07-19'
method: generated
source: openapi/laka-platform-openapi.json
operations:
- ReportingController_getPermittedPolicies
- ReportingController_getPermittedClaims
- ReportingController_getPermittedPoliciesWithInvoices
- PolicyController_getPolicies
- ClaimsController_findAll
- PolicyController_getAllDocuments
- PolicyController_getPolicyDocument
- PolicyController_getPolicyWording
- PolicyController_getPolicyIpid
- TasksController_getTasksByPerson
---

# Sync Laka data into your own system

**Laka publishes no webhooks and no event stream.** There is no AsyncAPI, no callback registration and
nothing that will tell you asynchronously that a policy lapsed or a claim settled. Polling the
reporting endpoints is the supported integration pattern, and this skill is how to do it without
hammering an API that publishes no rate limits.

## The three reporting endpoints

| Operation | Returns |
|---|---|
| `ReportingController_getPermittedPolicies` | All policies you are permitted to view |
| `ReportingController_getPermittedClaims` | All claims you are permitted to view |
| `ReportingController_getPermittedPoliciesWithInvoices` | All invoices |

These are the only Laka operations with real pagination. The general collection endpoints
(`PolicyController_getPolicies`, `ClaimsController_findAll`) do not declare pagination parameters in
the published spec — do not build a sync on them.

## Pagination and incremental filters

Query parameters available on the reporting surface:

- `page`, `pageSize`, `current` — page-number pagination.
- `updatedFrom`, `updatedTo` — ISO 8601 timestamp window. This is what makes the sync incremental.
- `useNewUpdatedPaging` — an opt-in flag toggling the newer updated-timestamp paging behaviour.
- `id`, `policyDetailId` — narrow to a specific record.

## The sync loop

1. Store a high-water mark: the `updatedTo` you last successfully processed.
2. Each run, request `updatedFrom = <high-water mark>` and `updatedTo = now`. Do **not** leave
   `updatedTo` open — pinning it makes the page set stable while you walk it, so records that change
   mid-walk do not shuffle between pages.
3. Walk `page` from 1 with a fixed `pageSize` until you get a short or empty page.
4. Only advance the high-water mark after the **whole** window has been processed. If page 4 of 9
   fails, re-run the entire window rather than resuming — these endpoints give you no cursor, so a
   partial resume is not safe.
5. Overlap the window by a small margin (re-request the last few minutes) and upsert by `id`. Records
   are UUID-keyed, so re-processing is harmless if your writes are upserts. This covers clock skew and
   any record whose `updatedAt` lands on the boundary.
6. Reconcile against your own records using `externalId` — the identifier you supplied, which Laka
   stores and returns on most responses.

## Sensible cadence

Laka documents no rate limits, no quota headers and no 429 response. Treat that as *unknown*, not as
*unlimited*: poll on a modest interval (minutes, not seconds), use a fixed `pageSize`, and back off
exponentially on any 5xx. There is no `Retry-After` to read, so choose your own ceiling.

## Documents

For each policy you sync, documents are fetched as **pre-signed URLs**, not inline binaries:

- `PolicyController_getAllDocuments` — every document for a policy and person in one call: policy
  document, insurance sticker, wording and IPID.
- `PolicyController_getPolicyDocument`, `PolicyController_getPolicyWording`,
  `PolicyController_getPolicyIpid` — individually.

Presigned URLs expire. Fetch the file and store it yourself; do not persist the URL and expect it to
resolve later.

## Outstanding work

`TasksController_getTasksByPerson` returns all tasks for a person, most recent first — use it to
surface what a member still needs to do rather than inferring it from policy state.

## Cautions

- No cursor pagination means no safe partial resume — window, then walk, then commit.
- Error responses are undocumented on the Platform API (the published OpenAPI declares success
  responses only). Treat any non-2xx as retryable-with-backoff and log the raw body.
- Send `x-api-key`, `x-api-region` and `x-api-language` on every request, against the region-correct
  host. A UK sync and an EU sync are two separate loops against two separate hosts.
