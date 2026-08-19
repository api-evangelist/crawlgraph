---
generated: '2026-08-13'
name: Compare link observations across Common Crawl releases
method: generated
description: Use the CrawlGraph changes endpoint to diff a domain's observed referring domains between two indexed Common Crawl snapshots — additions, absences, and authority movement.
api: openapi/crawlgraph-v1-openapi.yml
operations: [v1_list_releases_api_v1_releases_get, v1_changes_api_v1_changes_get]
source: >-
  Grounded in the live OpenAPI 3.1.0 at https://crawlgraph.com/api/v1/openapi.json (verbatim copy
  at openapi/_original/crawlgraph-openapi.json). Both operationIds verified verbatim in the spec.
  Response semantics and caveats per https://crawlgraph.com/docs/api section 6.
---

# Compare link observations across Common Crawl releases

Diff one domain's inbound-link picture between two indexed snapshots: which referring domains appeared, which are no longer observed, and whose authority moved.

This is the newest REST capability and it is **REST-only** — there is no MCP tool for it. An agent working over MCP alone cannot reach this. See `mcp/crawlgraph-tool-crosswalk.yml`.

## Auth and cost

- `Authorization: Bearer cg_live_<key>`.
- Costs **one call against the backlinks quota** — the same bucket as `POST /api/v1/backlinks`, not a separate one.

## Steps

1. **List queryable releases** — `v1_list_releases_api_v1_releases_get` (`GET /api/v1/releases`). Free. You need two releases with `available: true` for a comparison to be possible.

2. **Run the comparison** — `v1_changes_api_v1_changes_get` (`GET /api/v1/changes`). Query parameters:
   - `domain` (required, max 253 chars) — a strict domain. Schemes, paths, bare TLDs and IPs are rejected.
   - `from` (optional, max 64 chars) — the older release id.
   - `to` (optional, max 64 chars) — the newer release id.

   **Omit both** to get the newest queryable pair, which is what you want most of the time. If you pass `to` without `from`, its nearest queryable ancestor is used as `from`.

3. **Branch on `comparison_available` before reading anything else.** The 200 response is one of two shapes:
   - `V1ChangesAvailableResponse` — `comparison_available: true`, both `from_release` and `to_release` populated.
   - `V1ChangesUnavailableResponse` — `comparison_available: false`, `from_release: null`, all arrays empty, plus a `message` ("both indexed release artifacts are required for a comparison").

   **The unavailable shape is a successful 200 and it still consumes one backlinks call.** Do not treat it as an error and retry — you will just spend quota again. This differs from the single-release `409 release_unavailable`, which is free.

4. **Read the diff.**
   - `counts` — `{ from_snapshot, to_snapshot, added, removed, authority_moved }`.
   - `added[]` / `removed[]` — `V1ChangesObservedDomain`: `linking_domain`, `num_hosts`, `cg_authority`.
   - `authority_moved[]` — `V1ChangesAuthorityMovement`: `linking_domain`, `from_authority`, `to_authority`, `delta`.
   - `truncated` / `cap` — each snapshot is capped at **100,000** observed referring domains. If `truncated` is true, the counts and arrays describe only the capped comparison, so do not present them as complete totals.

## The caveat is mandatory, not optional

The response ships a `snapshot_caveat` string for a reason. Common Crawl snapshots are **periodic observations, not live link monitoring**.

- A domain in `removed[]` means *it was not observed in the newer snapshot*. It does **not** mean the page removed the link. Crawl coverage varies between releases.
- A domain in `added[]` may have been linking for a long time and simply been crawled this cycle for the first time.

Word every finding as "observed in / not observed in", never as "gained a link" or "lost a link". If you are asked to monitor links within days, say plainly that this API cannot do it and a continuous-crawl tool is the right instrument — the provider's own docs say the same.

## Errors

- `400 validation_error` — unknown release ids, **equal `from` and `to` ids**, or a malformed domain. Free; no quota consumed.
- `401 auth_missing` / `auth_invalid`.
- `429 quota_exceeded` — shares the backlinks bucket; read `Retry-After`.
- `500 internal_error` — quote the `request_id`.

Full catalog in `errors/crawlgraph-problem-types.yml`.
