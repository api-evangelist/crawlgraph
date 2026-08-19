---
generated: '2026-08-13'
name: Run a competitor backlink gap analysis
method: generated
description: Submit an async CrawlGraph gap-analysis job to find domains linking to your competitors but not to you, poll it to completion, and score the results into an outreach list.
api: openapi/crawlgraph-v1-openapi.yml
operations: [v1_gap_submit_api_v1_gap_analysis_post, v1_gap_poll_api_v1_gap_analysis__job_id__get, v1_lookup_backlinks_api_v1_backlinks_post]
source: >-
  Grounded in the live OpenAPI 3.1.0 at https://crawlgraph.com/api/v1/openapi.json (verbatim copy
  at openapi/_original/crawlgraph-openapi.json). All three operationIds verified verbatim in the
  spec. Async contract, quotas and error codes per https://crawlgraph.com/docs/api.
---

# Run a competitor backlink gap analysis

Find the domains that link to your competitors but not to you. That overlap is the highest-value prospect list in link building. This is a **two-step async flow** on the REST surface.

## Prerequisites

- A **lifetime-tier** key (`cg_live_...`). Gap analysis is **not** available on the free tier — a free key gets rejected here.
- Quota: 50 gap jobs per calendar month. See `rate-limits/crawlgraph-rate-limits.yml`.

## Steps

1. **Submit the job** — `v1_gap_submit_api_v1_gap_analysis_post` (`POST /api/v1/gap-analysis`). Body is `V1GapBody`:
   - `my_domain` (required) — your domain.
   - `competitor_domains` (required) — an array of **1 to 5** domains. More than 5 is a `400 validation_error`.

   Returns **202** with `V1GapSubmitResponse`: `{ job_id, status, poll_url }`. Capture `job_id` (prefixed `gap_`).

2. **Poll** — `v1_gap_poll_api_v1_gap_analysis__job_id__get` (`GET /api/v1/gap-analysis/{job_id}`). Returns `V1GapStatusResponse`. Loop with a **5-second sleep** until `status` is `completed` or `failed`. Typical completion is 5-30 seconds. `status` moves through `queued` → `running` → `completed` | `failed`, and `progress_pct` is populated while running.

3. **Read the result.** On `completed`, `result` holds a `V1GapResultBody`: `my_domain`, `competitor_domains`, `total_gaps`, and `gaps[]`. Each `V1GapResultGap` is `{ linking_domain, found_on[] }`, where `found_on` lists **which of your competitors that domain links to**. Check `truncated`.

4. **Score the shortlist** — gaps carry **no authority score**. To rank them, call `v1_lookup_backlinks_api_v1_backlinks_post` on the individual targets you care about and read `cg_authority`. Each such call spends one backlinks call, so enrich a top slice, not the whole list.

## The ranking rule that makes this useful

Sort by `len(found_on)` descending, **then** by authority.

A domain linking to *one* competitor might be a fluke or a paid placement. A domain linking to *three* of your competitors is a publisher who covers your whole category and has simply never heard of you. That overlap is the qualifier — use 2-3 competitors rather than 1 so the signal exists at all.

Filter out platform and CDN noise (`amazonaws.com`, `github.io`, `facebook.com`, and similar) before presenting the list; they are graph artifacts, not pitchable publishers.

## Quota discipline — read this before retrying

- Submitting spends a gap job **at submission**, not at completion.
- **Failed jobs do not refund quota in v1.** Do not build a naive retry loop around step 1.
- Polling (step 2) is free and safe to repeat.
- There is no idempotency key. A resubmitted job is a *new* job and a *new* charge — dedupe on your side by caching `job_id` against the `(my_domain, competitor_domains)` tuple. See `conventions/crawlgraph-conventions.yml`.
- Jobs are retained **7 days**; fetch the result before then.

## Errors

- `401 auth_missing` / `auth_invalid` — bad or missing bearer key.
- `400 validation_error` — malformed domain, or more than 5 competitors.
- `429 quota_exceeded` — monthly gap quota exhausted; read `Retry-After`.
- `404 not_found` on poll — the job does not exist **or is not yours**. The two are deliberately indistinguishable so job ids cannot be enumerated. Do not treat a 404 as "still queued" and keep polling.
- On `status: failed`, `error` carries `{ code, message }`. Quote the `request_id` to support.

Full catalog in `errors/crawlgraph-problem-types.yml`.

## Same flow over MCP — much shorter

The hosted MCP server at `https://crawlgraph.com/mcp` collapses steps 1-2 into a single `gap_analysis` tool call that submits and polls internally, and offers `gap_outreach_targets`, which additionally does step 4's ranking, the overlap tiering and the platform filtering for you. If you are an agent with MCP access, prefer those tools — see `skills/crawlgraph-find-warm-outreach-targets.md` and `mcp/crawlgraph-tool-crosswalk.yml`.
