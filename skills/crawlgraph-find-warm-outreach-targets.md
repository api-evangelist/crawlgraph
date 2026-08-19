---
generated: '2026-08-13'
name: Find warm outreach targets over MCP
method: generated
description: Use the hosted CrawlGraph MCP server to find the publishers that link to every one of your competitors but not to you, ranked and de-noised, then draft outreach against them.
api: mcp/crawlgraph-mcp.yml
mcp_endpoint: https://crawlgraph.com/mcp
tools: [gap_outreach_targets, backlinks, releases]
operations: [v1_gap_submit_api_v1_gap_analysis_post, v1_gap_poll_api_v1_gap_analysis__job_id__get, v1_lookup_backlinks_api_v1_backlinks_post]
source: >-
  Grounded in the live MCP tools/list probed anonymously at https://crawlgraph.com/mcp on
  2026-08-13 (verbatim result at mcp/crawlgraph-mcp-tools.json) — every tool name, parameter and
  constraint below is read from the published inputSchema. Backing REST operationIds per
  mcp/crawlgraph-tool-crosswalk.yml.
---

# Find warm outreach targets over MCP

The highest-leverage CrawlGraph flow, and the one that is genuinely easier on the agent surface than on REST. `gap_outreach_targets` is a composite tool: it runs the gap job, polls it, tiers the results by competitor overlap, strips platform noise, and authority-scores the top targets — work a REST caller has to build by hand.

## Connect

Hosted (no install):

```
claude mcp add --scope local --transport http \
  crawlgraph https://crawlgraph.com/mcp \
  --header='Authorization: Bearer cg_live_<your-key>'
```

Or for Codex CLI:

```
export CRAWLGRAPH_API_KEY="cg_live_<your-key>"
codex mcp add crawlgraph --url https://crawlgraph.com/mcp --bearer-token-env-var CRAWLGRAPH_API_KEY
```

Local stdio fallback for clients without remote-server support: `npx -y crawlgraph-mcp` with `CRAWLGRAPH_API_KEY` in the env. Note the npm package is pinned at 0.1.0 (published 2026-05-31) and lags the repository — prefer the hosted endpoint. See `packages/crawlgraph-packages.yml`.

Requires the **lifetime** tier. `tools/list` answers anonymously, but calling these tools does not.

## Steps

1. **`gap_outreach_targets`** — the whole play in one call.
   - `my_domain` (required).
   - `competitor_domains` (required) — **2 to 5** domains. Note the floor is 2 here, not 1 as on `gap_analysis`: the overlap signal does not exist with a single competitor.
   - `include_platforms` (optional, default `false`) — leave it false. True re-admits CDN/social/platform domains, which are graph artifacts, not publishers you can pitch.
   - `enrich_top` (optional, default 10, max 25) — how many priority targets get authority-scored. **Each one costs a backlinks call**, so raise it deliberately. `0` disables enrichment.

2. **Read the two tiers** from the output.
   - `priority_targets[]` — domains linking to **all** your competitors but not you. These are publishers who cover your entire category and have never heard of you. This is the list.
   - `secondary_targets[]` — domains linking to 2+ competitors. Your second wave.
   - Each carries `linking_domain`, `found_on[]`, `overlap`, and (when enriched) `cg_authority` / `cg_rank`.
   - `total_gaps`, `platforms_filtered` and `authority_enriched` tell you how much was cut and how much was scored.

3. **Score more, if needed** — call the `backlinks` tool on any individual target to get its `cg_authority`. Use `releases` first if you need to pin a specific Common Crawl snapshot.

4. **Draft the outreach.** Each target's `found_on` names the competitors it already covers — that is the pitch. The case is "you already cover this space, here is the one you are missing", which is the easiest version of this email to write.

## Costs

One `gap_outreach_targets` call spends **one gap job plus one backlinks call per enriched target**. With the default `enrich_top: 10` that is 1 gap job + 10 backlinks calls against a monthly allowance of 50 and 1,000. Budget accordingly; the tool does not warn you before spending.

## What the tool will not do

- It cannot find contact emails. The docs note lifetime *dashboard* users get a contact email per target; the API and MCP surfaces return domains only.
- It has no idempotency. Re-running the same analysis is a second charge.
- It has no access to `GET /api/v1/changes` — cross-release change tracking is REST-only and has no MCP tool. See `mcp/crawlgraph-tool-crosswalk.yml`.

## Caveat to carry into any answer

The underlying data is a **quarterly Common Crawl snapshot**, not live monitoring. A domain in the gap list was observed linking to your competitor in a named release; it is not proof the link is live today. Present findings as prospects, never as a current link audit.

## Errors

The MCP endpoint surfaces the REST error envelope. `401` means the bearer key is missing, invalid or revoked. `405` means the client sent a browser-style GET instead of an MCP POST. `406` means the client omitted the `Accept: application/json, text/event-stream` header. A `429` is a quota exhaustion shared with the HTTP API. See `errors/crawlgraph-problem-types.yml`.
