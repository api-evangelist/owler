---
name: Map a company's competitive landscape with Owler
description: Pull Owler's crowdsourced, scored competitor list for a company and expand each
  competitor into a full company record to build a ranked landscape.
api: openapi/owler-enterprise-api-openapi.yml
operations:
  - getCompetitorsForWebsite
  - getCompetitorsForId
  - getCompanyById
generated: '2026-08-14'
method: generated
source: openapi/_original/owler-enterprise-api-openapi.json
---

# Map a company's competitive landscape with Owler

Owler's competitive graph is the thing Owler actually sells: competitor relationships nominated and
validated by its user community, each carrying a numeric `score`. This skill turns one company into
a ranked landscape.

## Before you start

- Base URL `https://apiv2.owler.com`, header `x-api-key: <your Owler API key>`.
- Competitor Premium is separately licensed from Company Premium. A key that works for one may
  return `403` on the other, and the contract gives you no way to tell that apart from a bad key.

## Steps

1. **Get the competitor list.** If you have a domain, call `getCompetitorsForWebsite`:
   `GET /v1/company/competitorpremium/url/{website}`. If you already hold an Owler id, call
   `getCompetitorsForId`: `GET /v1/company/competitorpremium/id/{companyId}`. Add `?format=json`.

2. **Send the right first-page cursor.** On the competitor operations, `pagination_id` must be
   `*` on the FIRST request. (The feed operations use a blank value instead — do not share cursor
   code between the two families.)

3. **Page until you have the whole list.** Each response is a `competitors` envelope:
   `competitor[]` plus a `pagination_id`. Pass that `pagination_id` back on the next request.
   Owler does not document what the field contains on the final page, so guard your loop: stop
   when the array comes back empty or when `pagination_id` stops changing, and cap the number of
   iterations.

4. **Rank by `score`.** Every `CompetitorBasicVO` carries `company_id`, `name`, `short_name`,
   `website`, `logo_url`, `profile_url` and `score`. `score` is the competitive-strength signal and
   it exists nowhere else in the contract — it is the reason to call this endpoint rather than
   inferring competitors from industry codes.

5. **Expand only what you need.** The competitor objects are stubs. To get revenue, employees,
   funding or CEO for each one, call `getCompanyById` per competitor
   (`GET /v1/companypremium/id/{companyId}`). There is no batch company lookup, so a 25-competitor
   landscape costs 25 additional calls. Rank first, expand the top N.

## Error handling

- `403` — bad key or no Competitor Premium entitlement. Stop and escalate; do not retry in a loop.
- `404` — the subject company is not in Owler's graph.
- `429` — back off exponentially with jitter; no limit and no `Retry-After` are published. This is
  the operation most likely to hit it, because expansion fans out.
- `400` — check `format` and that the path identifier is present.
- Error bodies have no documented schema on any status. Branch on the status code.

## Do not

- Do not assume the list is exhaustive or symmetric. It is crowdsourced: if A lists B, B does not
  necessarily list A.
- Do not expand every competitor by reflex — that is the fastest way to a 429 against an
  unpublished allowance.
