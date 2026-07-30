---
name: laka-file-a-claim
description: Create, evidence and submit an insurance claim against a Laka policy — including the
  two-step presigned attachment upload and the claim status model.
api: Laka Platform API
generated: '2026-07-19'
method: generated
source: openapi/laka-platform-openapi.json, https://docs.laka.co/docs/introduction
operations:
- ClaimsController_create
- ClaimsController_findAll
- ClaimsController_findOne
- ClaimsController_update
- ClaimsController_createClaimAttachmentUploadLink
- ClaimsController_createClaimAttachment
- ClaimsController_updateClaimAttachmentStatus
- ClaimsController_addNoteToClaim
- ClaimsController_submitClaim
- FleetController_createFleetClaim
---

# File a claim with Laka

Use this when a rider on your Group policy has had a bike stolen or damaged and you are submitting the
claim on their behalf through the Laka Platform API.

## Before you call anything

- Regional base URL `https://api-{region}.app.laka.co` (`gb` or `nl`); test on
  `https://api-{region}.staging.laka.co`.
- Auth header `x-api-key`, plus the required `x-api-region` and `x-api-language` headers.
- Claims operations live under `/v3/claims`.

## The shape of a claim

A `Claim` carries `id`, `personId`, `policyId`, `productIds[]`, `claimType`, `claimStatus`,
`incidentDate`, `incidentDetails[]`, `address` (the **incident** address — where it happened, *not*
the policyholder's home address), `crimeReferenceNumber`, a human-readable `claimReference` (e.g.
`C-12345`) and `createdAt`/`updatedAt`.

`claimType` is one of: `theft`, `damage`, `health`, `liability`.

`claimStatus` moves through: `Created` → `Incomplete` → `Submitted` → `In Progress` → `On Hold` →
`Underwriter Review` → `Settled` / `Rejected` / `Closed`.

Required on the entity: `id`, `personId`, `createdAt`, `updatedAt`, `claimReference`, `productIds`.

## Steps

1. **Create the claim.** `ClaimsController_create` opens a claim against a policy. It lands in
   `Created`. Set `claimType`, `incidentDate`, `policyId` and the `productIds` being claimed against.
   `address.country` is **required** when creating or submitting — it accepts an ISO 3166-1 alpha-2
   code (`GB`) or a full country name (`United Kingdom`).

2. **Fill in the detail.** `ClaimsController_update` patches the claim as the rider supplies more —
   incident details, crime reference number for a theft, corrected incident address. Read the current
   state back with `ClaimsController_findOne`.

3. **Attach evidence — this is a three-call flow, not an upload.**

   a. `ClaimsController_createClaimAttachmentUploadLink` returns a **pre-signed URL**.
   b. `PUT` the raw binary file data directly to that URL. This is a plain HTTP PUT to storage — do
      not send your Laka API key to it.
   c. `ClaimsController_createClaimAttachment` confirms the upload. Laka is explicit that this must
      be called *after* the file has reached the presigned URL.

   Then move the attachment through its states with
   `ClaimsController_updateClaimAttachmentStatus`.

   If you skip step (c), the file exists in storage but the claim does not know about it — a silent
   failure mode worth asserting against in tests.

4. **Add context for the handler.** `ClaimsController_addNoteToClaim` creates an **internal** note
   that appears on the claim. Internal means Laka's handlers see it; do not put anything in it you
   would not want an assessor reading.

5. **Submit.** `ClaimsController_submitClaim` submits the claim for processing. Laka states it should
   be called only **after all necessary information has been provided** — submitting an incomplete
   claim is what produces the `Incomplete` status and a round trip. Validate that you have the
   incident address with country, the incident date, the claim type, the product ids and any crime
   reference before you call it.

6. **Track it.** `ClaimsController_findOne` for one claim, `ClaimsController_findAll` for every claim
   you are permitted to view. For bulk reconciliation use
   `ReportingController_getPermittedClaims`, which supports `updatedFrom`/`updatedTo` paging — see the
   `laka-reporting-sync` skill.

## Fleet claims

`FleetController_createFleetClaim` creates a claim against a fleet product. Use this rather than the
standard claim create when the insured item belongs to a fleet arrangement.

## Cautions

- **No idempotency keys.** Retrying `ClaimsController_create` opens a second claim against the same
  incident. Before any write retry, re-read with `ClaimsController_findOne` or search
  `ClaimsController_findAll`.
- **`ClaimsController_submitClaim` is not reversible through the API** — there is no unsubmit
  operation. Get the claim right first.
- **No webhooks.** You will not be notified when a claim moves to `Settled` or `Rejected`. Poll
  `ReportingController_getPermittedClaims` with `updatedFrom`.
- **Error responses are undocumented** in the published Platform OpenAPI (success responses only).
  Handle non-2xx defensively and log raw bodies.
- Claims contain personal and incident data. Send them to the region-correct host with the matching
  `x-api-region` — that header is how Laka attributes the insured party to the right regulatory
  region.
