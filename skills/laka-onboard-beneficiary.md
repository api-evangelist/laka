---
name: laka-onboard-beneficiary
description: Add a member to a Laka Group policy as a beneficiary and attach their insured products,
  using the Laka Platform API — including the single-call combined relationship path and how to end
  the relationship cleanly.
api: Laka Platform API
generated: '2026-07-19'
method: generated
source: openapi/laka-platform-openapi.json, https://docs.laka.co/docs/introduction
operations:
- PolicyController_getPolicies
- PolicyController_getPolicyById
- PolicyController_getPackage
- PersonController_create
- PersonController_findByExternalId
- PersonController_findByEmail
- PersonController_findOne
- PersonController_patchPerson
- PersonController_createAccountRelationship
- PolicyController_createCombinedRelationship
- PolicyController_postPolicyProducts
- PolicyController_patchPolicyProducts
- PolicyController_uploadPolicyProductsCsv
- PolicyController_deletePolicyProduct
- PolicyController_endPolicyRelationship
- PolicyController_requestCover
---

# Onboard a beneficiary onto a Laka Group policy

This is the **Group partner** flow. Laka has issued you a Group insurance policy; you extend its
benefits to your members and customers on your own terms. Your job is to put people on the policy,
attach the gear they are insuring, and take them off again when they leave.

## Before you call anything

- **Base URL is regional.** `https://api-{region}.app.laka.co` where region is `gb` or `nl`. Test
  against `https://api-{region}.staging.laka.co`.
- **Auth header is `x-api-key`** on the Platform API (the Quote API uses `Authorization` instead).
- **`x-api-region` and `x-api-language` are required on every request.**
- Platform operations live under the `/v3` path prefix.

## Steps

1. **Know which policy you are working against.** `PolicyController_getPolicies` returns every policy
   you are permitted to view; `PolicyController_getPolicyById` fetches one. If you are working from a
   package, `PolicyController_getPackage` returns the package detail.

2. **Find the person before you create them.** Look them up by the identifier you already hold:
   - `PersonController_findByExternalId` — your own identifier (CRM ID, internal user ID). This is
     the one to prefer, because you control it.
   - `PersonController_findByEmail` — email lookup.
   - `PersonController_findOne` — by Laka `personId` (UUID).

   Always set `externalId` when you create a person. Laka stores it and returns it on most responses,
   which is what makes reconciliation against your own system possible later.

3. **Create or update the person.** `PersonController_create` creates a person record;
   `PersonController_patchPerson` updates one. `type` is one of `Person`, `Organisation` or `Rider`.
   `id`, `type` and `email` are required on the Person entity.

4. **Attach them to the policy.** You have two routes:

   - **Preferred — one call.** `PolicyController_createCombinedRelationship` creates or updates the
     person, the relationship and the policy products in a single request. Laka documents this
     explicitly as the endpoint to use "in place of performing the create operations individually."
     Use it: fewer calls means fewer half-finished states, which matters because Laka has no
     idempotency keys and no transactional rollback across separate calls.
   - **Step by step.** `PersonController_createAccountRelationship` creates the account/person
     relationship, then `PolicyController_postPolicyProducts` creates the products and associates them
     with the policy.

5. **Manage their gear.** `PolicyController_patchPolicyProducts` updates products on the policy.
   `PolicyController_deletePolicyProduct` deletes a product by UUID and dis-associates it from the
   policy. For onboarding many members at once,
   `PolicyController_uploadPolicyProductsCsv` creates products in bulk from CSV — it returns per-row
   results (`RowSuccess` / `RowError`), so **always read the row errors**; a 2xx on the upload does
   not mean every row landed.

6. **Off-boarding.** `PolicyController_endPolicyRelationship` ends the relationship between a person
   and the policy — the person and their gear are removed and their subscription ends. This is the
   correct call when a member leaves; do not simply delete their products, which leaves the
   relationship dangling.

## Requesting cover directly

`PolicyController_requestCover` creates and activates policies from quote requirements. Use this when
you have already resolved requirements through the Quote API and want cover created without a rider
handoff.

## Reconciliation

Poll the reporting surface rather than diffing state yourself — see the `laka-reporting-sync` skill.
`ReportingController_getPermittedPolicies` supports `updatedFrom` / `updatedTo` so you can pull only
what changed.

## Cautions

- **No idempotency.** Laka publishes no idempotency key header and no replay semantics. A retried
  `PolicyController_postPolicyProducts` can double-add products. Before retrying any write, re-read
  with `PolicyController_getPolicyById` and confirm what actually landed.
- **Error responses are undocumented.** The published Platform OpenAPI declares success responses only
  (200/201/204). Do not assume a structured error envelope — handle non-2xx defensively and log the
  raw body.
- **No webhooks.** Nothing tells you asynchronously that a policy changed. Poll the reporting
  endpoints.
- **Identifiers are UUID v4 with no type prefix**, so a policyId and a personId are visually
  indistinguishable. Keep them typed in your own code.
