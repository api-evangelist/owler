---
name: Monitor company news and funding events with Owler
description: Poll Owler's Feed API for news, press, funding, acquisition, people, blog and video
  items across a watchlist of up to 10 companies per call, with category filtering and cursor
  paging.
api: openapi/owler-enterprise-api-openapi.yml
operations:
  - getFeeds
  - getFeedsByWebsite
generated: '2026-08-14'
method: generated
source: openapi/_original/owler-enterprise-api-openapi.json
---

# Monitor company news and funding events with Owler

The Feed API is Owler's content-monitoring product. It is the only batch surface in the contract
and the only way an application gets freshness — Owler ships no webhooks, so monitoring is
poll-only.

## Before you start

- Base URL `https://apiv2.owler.com`, header `x-api-key: <your Owler API key>`.
- Two entry points, same envelope: `getFeeds` (`GET /v1/feed`, keyed by `company_id`) and
  `getFeedsByWebsite` (`GET /v1/feed/url`, keyed by `domain`).
- Prefer `getFeeds` with Owler ids once you have them — ids are unambiguous, domains are not.

## Steps

1. **Chunk your watchlist into groups of 10.** Both operations accept a comma-separated list and
   cap it at 10 identifiers (`company_id` on `getFeeds`, `domain` on `getFeedsByWebsite`). A
   200-company watchlist is 20 calls per sweep, minimum.

2. **Filter by category at the source.** `category` accepts a comma-separated subset of
   `NEWS`, `PRESS`, `FUNDING`, `ACQUISITION`, `PEOPLE`, `BLOG`, `VIDEOS`. Omitting it returns every
   category. For a funding-signal workflow send `category=FUNDING,ACQUISITION` — filtering
   server-side is free, filtering client-side costs you page quota.

3. **Set `limit` deliberately.** Default 10, maximum 100. Use 100 for a backfill sweep and a small
   value for a tight polling loop. Note the parameter is typed as a string in the spec.

4. **Page with the BLANK first-page cursor.** On the feed operations `pagination_id` must be blank
   on the first request — NOT `*`, which is what the competitor operations require. Take
   `pagination_id` off the response envelope and pass it back on the next call.

5. **Deduplicate on `FeedsVO.id`.** Because monitoring is poll-based, consecutive sweeps will
   overlap. `id` is the stable item key. Also persist the highest `feed_date` you have processed per
   company so you can stop paging early on the next sweep — but note `feed_date` is a bare string
   with no documented format, so normalize it once at ingest rather than comparing raw.

6. **Use the embedded company stub.** Every item carries `company` (a `CompanyBasicVO` with
   `company_id`, `name`, `short_name`, `website`, `logo_url`, `profile_url`), so you can attribute a
   batched response back to the right watchlist entry without a second lookup. Items also carry
   `title`, `description`, `source_url` (the publisher), `owler_feed_url` (Owler's own permalink),
   `publisher_name`, `publisher_logo` and `enclosure_image`.

## Polling cadence

There is no published rate limit, no `Retry-After` and no webhook alternative, so pick a
conservative sweep interval and let 429s widen it:

- On `429`, back off exponentially with jitter and reduce concurrency for the rest of the sweep.
- On `403`, stop the whole sweep — either the key is wrong or the account is not licensed for
  content monitoring.
- On `500`, retry the chunk with backoff (GETs are safe) and check https://status.owler.com/ if it
  persists.
- Note the feed operations do NOT declare `404`; an unknown company yields an empty `feed[]`
  rather than an error, so an empty result is not proof of a bad identifier.

## Do not

- Do not expect push. Owler's "instant alerts" go to humans in the app, email, Slack and Teams —
  there is no webhook, no callback and no AsyncAPI for applications.
- Do not exceed 10 identifiers per call hoping it will be truncated silently; the documented cap is
  a `400`.
