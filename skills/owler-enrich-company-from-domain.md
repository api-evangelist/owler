---
name: Enrich a company from its domain with Owler
description: Turn a bare company website domain into Owler's full company record — firmographics,
  funding history, acquisitions, CEO and stock listing — and resolve the Owler company id for
  follow-on calls.
api: openapi/owler-enterprise-api-openapi.yml
operations:
  - getCompanyByWebsite
  - getCompanyById
generated: '2026-08-14'
method: generated
source: openapi/_original/owler-enterprise-api-openapi.json
---

# Enrich a company from its domain with Owler

Use this when you hold a website domain (from a CRM record, an email address, a signup form) and
need Owler's structured company record.

## Before you start

- Base URL: `https://apiv2.owler.com`
- Every request needs the header `x-api-key: <your Owler API key>`. Keys come from the Owler
  data-licensing sales process — there is no self-serve signup and no test key.
- Everything here is a GET. Retries are always safe.

## Steps

1. **Look the company up by domain.** Call `getCompanyByWebsite`:
   `GET /v1/companypremium/url/{website}` with the bare domain as the path segment (for example
   `stripe.com`, not `https://www.stripe.com/`). Add `?format=json` explicitly — Owler selects the
   response format with a query parameter, not an Accept header.

2. **Handle the lookup outcome before parsing.**
   - `200` — parse the `companyPremium` object.
   - `404 Resource Not Found` — the domain is not in Owler's graph. Retry once with the apex form
     if you sent a subdomain (or vice versa) before giving up. Do not retry the identical request.
   - `403 Authentication Failed` — either the key is wrong OR the key is not licensed for Company
     Premium. Owler declares no 401, so you cannot tell these apart from the response. Stop and
     escalate; never loop.
   - `429` — back off exponentially with jitter. Owler publishes no limit and no `Retry-After`, so
     choose your own floor and treat the ceiling as unknown.
   - `400` — you sent a bad `format` value or a malformed path. Fix it; it will not succeed on
     retry unchanged.

3. **Persist `company_id` immediately.** It is the integer key every other Owler product accepts,
   and holding it lets you skip the domain round-trip next time. From then on use `getCompanyById`
   (`GET /v1/companypremium/id/{companyId}`) — same record, keyed by id.

4. **Read the record defensively.** No schema in Owler's contract declares `required`, a `format`
   or an example. In particular:
   - `founded_date` and all `Funding.date` / `Acquisition.date` values are bare strings with no
     documented format.
   - `revenue`, `employee_count`, `Funding.amount` and `Acquisition.amount` are strings, not
     numbers.
   - `stock` is only meaningful for public companies — check `company_type` first.
   - `Investor.company_id` is a STRING while `companyPremium.company_id` is an INTEGER. Coerce
     before comparing them.

5. **Walk the graph if you need it.** `portfolio_company_ids[]` holds bare ids of portfolio
   companies and `acquisition[]` carries both `company_id` and `acquirer_company_id`. Each one
   costs a separate `getCompanyById` call — there is no batch company lookup and no expansion
   parameter. Budget the calls before you fan out.

## Do not

- Do not build a fixed-rate scheduler. The rate limit is enforced (429 is declared on every
  operation) but unpublished.
- Do not request `format=xml`. The parameter offers it, but the spec declares only
  `application/json` and the XML shape is undocumented.
- Do not send an `Idempotency-Key`. Owler has no such mechanism, and with a read-only API it is
  unnecessary.
- Do not call `api.owler.com`. That is the retired v1 host and it no longer completes a TLS
  handshake.
