---
generated: '2026-08-13'
name: Look up referring domains for a domain
method: generated
description: Get a free CrawlGraph API key, then pull the referring domains (backlinks) for any target domain from the Common Crawl webgraph, ranked by authority.
api: openapi/crawlgraph-v1-openapi.yml
operations: [request_free_key_api_v1_free_key_post, v1_list_releases_api_v1_releases_get, v1_lookup_backlinks_api_v1_backlinks_post]
source: >-
  Grounded in the live OpenAPI 3.1.0 at https://crawlgraph.com/api/v1/openapi.json (verbatim copy
  at openapi/_original/crawlgraph-openapi.json). All three operationIds verified verbatim in the
  spec. Quotas, headers and error codes per https://crawlgraph.com/docs/api, captured in
  rate-limits/ and errors/.
---

# Look up referring domains for a domain

The core CrawlGraph flow: given a domain, return every site in the Common Crawl hyperlink graph that links to it, with an authority score per linker.

## Auth

- Send `Authorization: Bearer cg_live_<key>`. Every `/api/v1/*` route requires it **except** the free-key route in step 1.
- Base URL: `https://crawlgraph.com` — paths already carry the `/api/v1` prefix.
- See `authentication/crawlgraph-authentication.yml`.

## Steps

1. **Get a key (once, optional if you already have one)** — `request_free_key_api_v1_free_key_post` (`POST /api/v1/free-key`). Unauthenticated. Body is `FreeKeyRequest`: `{ "email": "you@example.com" }`. The key is **emailed** — it is never returned in the response body, so do not parse the response looking for it. One active key per email; the free tier is 15 backlink calls per calendar month.

2. **Discover which snapshots you can query** — `v1_list_releases_api_v1_releases_get` (`GET /api/v1/releases`). Free, never counts against quota. Returns `V1ReleasesResponse`: `releases[]` of `{ id, label, available }`. **Only pass a `release_id` where `available` is `true`** — a known-but-unloaded release returns `409 release_unavailable`. Omit `release_id` entirely to get the latest.

3. **Run the lookup** — `v1_lookup_backlinks_api_v1_backlinks_post` (`POST /api/v1/backlinks`). Body is `V1BacklinksBody`:
   - `domain` (required) — a strict registrable domain like `example.com`. Schemes, paths, bare TLDs and IP addresses are rejected with `400 validation_error`.
   - `limit` (optional) — default 1000, **max 10000**. Out of range is a `400`.
   - `sort` (optional) — `authority` (default) or `hosts`.
   - `release_id` (optional) — from step 2.

4. **Read the response** (`V1BacklinksResponse`). `total_linking_domains` is the true total; `returned` is how many you actually got. If `returned < total_linking_domains` your result is truncated by `limit`. Each item in `results[]` is a `V1BacklinkItem`: `linking_domain`, `num_hosts`, `tld`, `cg_authority`, `cg_rank`.

## Interpreting the scores

- `cg_authority` — 0-100 log-rank percentile derived from Common Crawl harmonic centrality. Higher is more authoritative. Use it for **ordering a candidate list**, not as a prediction that a link will move rankings.
- `cg_rank` — raw PageRank position across the whole graph (1 = top-ranked domain).
- Both are `null` for domains absent from the ranks file. **Filter for `cg_authority != null` before sorting** or your nulls will pollute the ranking.
- `num_hosts` counts distinct hosts under that domain linking to the target — one referring domain can carry many links.

## Pacing

There is no pagination. Pace from the response headers, which are present on every 2xx:

- `X-RateLimit-Limit-Backlinks` / `X-RateLimit-Remaining-Backlinks` — your monthly cap and what is left.
- `X-RateLimit-Reset` — Unix timestamp of the next month rollover.
- Stop before `Remaining` hits 0 rather than discovering it via a `429`.

Only successful 2xx calls are charged. Validation errors, auth failures and quota rejections are free. See `rate-limits/crawlgraph-rate-limits.yml`.

## Errors

Every non-2xx is `{ "error": "<code>", "message": "...", "request_id": "req_..." }`. Handle:

- `401 auth_missing` / `auth_invalid` — fix the header, or the key is revoked.
- `400 validation_error` — bad domain, unknown `release_id`, or `limit` out of range.
- `409 release_unavailable` — re-read step 2 and pick an `available: true` release. Free; no quota consumed.
- `429 quota_exceeded` — read `Retry-After`.

Retry is safe on this operation — it is a read. Log `request_id` on every failure; it is the only support handle. Full catalog in `errors/crawlgraph-problem-types.yml`.

## Caveat to carry into any answer

CrawlGraph is a **quarterly Common Crawl snapshot**, not a live crawler. It is built for one-off prospecting, not link monitoring. Never report its results as "current backlinks" — report them as observed in the named `release_label`, which the response hands you for exactly this reason.

## Same flow over MCP

The hosted MCP server at `https://crawlgraph.com/mcp` exposes this as the `backlinks` tool with the same parameters and the same quota. See `mcp/crawlgraph-mcp.yml`.
