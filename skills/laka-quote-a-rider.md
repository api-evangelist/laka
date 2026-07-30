---
name: laka-quote-a-rider
description: Price and place bicycle insurance for a rider using the Laka Quote API — create a quote,
  resolve its data requirements, price it, hydrate it into policy steps, and hand the rider off into
  Laka onboarding.
api: Laka Quote API
generated: '2026-07-19'
method: generated
source: openapi/laka-quote-openapi.json, https://docs.laka.co/docs/introduction
operations:
- quote#getDataRequirements
- quote#new
- quote#quickQuote
- quote#get
- quote#patch
- quote#edit
- quote#validateDataRequirement
- quote#validationQuote
- quote#getPricing
- quote#getPricingComponents
- quote#hydrate
- quote#getSteps
- quote#handoff
- quote#start
---

# Quote a rider with Laka

This is the **Introducer partner** flow. Use it when you want to show a rider a price and send them
into Laka to buy cover, while keeping the regulatory burden on your side manageable. You are not
administering a policy here — you are producing a quote and handing off.

## Before you call anything

- **Base URL is regional.** `https://api-{region}.app.laka.co/v2/quotes` where region is `gb` (UK) or
  `nl` (EU). Use `https://api-{region}.staging.laka.co/v2/quotes` for testing — Laka explicitly
  directs partners to Staging.
- **Auth header is `Authorization`**, not `x-api-key`. The Quote API's key goes in `Authorization`;
  the Platform API's key goes in `x-api-key`. Mixing these up is the most common Laka integration
  error.
- **Always send `x-api-region` and `x-api-language`.** These are required. `x-api-region` attributes
  the insured party to the correct regulatory region; `x-api-language` sets the language Laka uses for
  any text it sends the customer. Belgium is the reason both exist separately — `be`/`nl` and `be`/`fr`
  are both valid.
- All traffic must be HTTPS.

## Steps

1. **Find out what you need to ask.** Call `quote#getDataRequirements` with the `packageId` you are
   quoting. This returns the `DataRequirement` set — the questions Laka needs answered before it can
   price and place cover. Do not hard-code a question list; it is package- and region-dependent.

2. **Optionally get a fast indicative price first.** `quote#quickQuote` returns a price without a
   persisted quote. Use it for "from £X/month" style surfaces where you do not yet want to create a
   quote record.

3. **Create the quote.** `quote#new` creates a persisted quote and returns its id. Everything after
   this point is keyed on that id.

4. **Fill it in.** Use `quote#patch` (partial) or `quote#edit` (full replace) to write the rider's
   answers and the insured products onto the quote. Read the current state back with `quote#get`
   whenever you need to reconcile.

5. **Validate as you go.** `quote#validateDataRequirement` checks a single response against one
   requirement — call it inline so the rider sees errors at the field, not at submit.
   `quote#setViewedDataRequirement` and `quote#getViewedDataRequirements` let you track which
   requirements the rider has actually been shown, which matters for the disclosure trail.

6. **Validate the whole quote.** `quote#validationQuote` validates the quote against all its attached
   data requirements. Do not proceed to pricing or handoff until this passes.

7. **Price it.** `quote#getPricing` returns pricing for the set of products.
   `quote#getPricingComponents` breaks the price down by component for a given `quoteId` — use it when
   you need to show the rider what they are paying for rather than a single number.

8. **Hydrate and inspect the steps.** `quote#hydrate` turns the quote into something that can become
   policies; `quote#getSteps` returns the steps required to create policies from a hydrated quote.
   Read the steps before handing off so you know what Laka will still ask the rider.

9. **Hand off.** `quote#handoff` sends the rider into Laka's onboarding with their information
   pre-filled. `quote#start` begins the quote journey. From here Laka owns the purchase, the
   regulatory disclosures and the policy.

## Cleaning up

- `quote#deleteProducts` removes products from a quote.
- `quote#deleteAccessoryChildren` removes accessory child items.

Use these rather than abandoning and recreating a quote — the quote carries the viewed-requirement
trail, and recreating it loses that.

## Errors

The Quote API returns a **Goa error envelope** under the `application/vnd.goa.error` media type — not
RFC 9457 `application/problem+json`. The envelope carries `fault` (true when it is a server-side
fault), `id` (a unique identifier for this occurrence — log it, it is the only trace handle Laka
publishes), `message`, `name`, `temporary` and `timeout`.

Documented statuses: `400` invalidArgument, `401` unauthorized, `404` notFound, `409` conflict,
`500` internalServerError.

Retry only when `temporary` or `timeout` is true. **Laka publishes no idempotency key and no
request-replay semantics** — so a blind retry of `quote#new` creates a second quote. Retry reads
freely; for writes, re-read with `quote#get` before retrying.

## What Laka does not give you here

No webhooks or events — you will not be told asynchronously that the rider bought cover. To check
whether a rider took out insurance you poll. No documented rate limits, so back off politely on 5xx
rather than assuming headroom.
