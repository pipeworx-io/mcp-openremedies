# OpenRemedies

OpenRemedies is a lookup of sourced treatment records (approved, off-label, supplements, in development, recruiting). It never recommends treatment, diagnoses, doses, or declares trial eligibility. Every record carries a citation; empty results mean no records in covered sources, not that no treatments exist. Built on pipeworx.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1875+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

*(That paragraph is the medical-content posture from the OpenRemedies MCP
endpoint's `initialize.instructions`, preserved verbatim per fleet #1392.
Keep it word-for-word in any public catalog entry for this pack.)*

This pack proxies to the OpenRemedies MCP endpoint (Bruce's free
treatment-lookup product, repo `b-gutman/openremedies`). It is keyless,
read-only public data; OpenRemedies applies its own rate limiting at its
Worker. Every response carries `meta` with source citations, per-source
freshness, `data_version`, and coverage notes.

## Tools

| Tool | What it returns |
|------|-----------------|
| `resolve_condition` | Condition name/lay term/misspelling → condition concepts (candidates, never merged) |
| `resolve_substance` | Drug/herb/supplement name → substance records with labeled salt/form family |
| `get_condition_options` | Five fixed sections for a condition: approved, documented off-label, herbs/supplements, in development, recruiting |
| `get_substance_uses` | Approvals by jurisdiction, verbatim indications, evidence assertions |
| `find_trials` | Registry trials by condition/substance/status/phase/country |
| `get_related_concepts` | Broader/subtype/sibling condition concepts, each labeled by its structural relationship (type of / subtype / related type). Derived from the disease hierarchy — not an assertion of clinical relatedness |
| `get_evidence` | One full evidence assertion with verbatim finding and provenance |
| `get_coverage` | What is and is NOT covered: per-source freshness, jurisdiction matrix, evidence counts |

Typical flow: `resolve_condition("migraine")` → `condition_id` →
`get_condition_options(condition_id)`; or `resolve_substance("sumatriptan")`
→ `substance_id` → `get_substance_uses(substance_id)`.

`get_related_concepts` takes the same `condition_id` and answers "what else
should I look at" — it returns an empty list for concepts with no hierarchy
neighbours carrying records (e.g. `psoriasis`, `atopic eczema`), which means
no neighbours in the covered hierarchy, not that none exist.

## Listed; origin flip still pending (fleet #3443)

Bruce lifted the unlisted hold for search and routing on 2026-10-21. The pack
is in the catalog, in `search_packs` and on routing menus. The origin in
`src/index.ts` still points at `staging.openremedies.com` because the apex is
not cut over; flipping `ORIGIN` to `https://openremedies.com/mcp` and adding a
registry-api `/uptime` row stay launch items for the OpenRemedies PM.

The tool definitions in `src/index.ts` are copied from the endpoint's
`tools/list`. If OpenRemedies changes its tool surface, update them to match;
nothing syncs them automatically. Coverage figures (sources, jurisdictions)
come from `get_coverage`; never hard-code them in a description.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "openremedies": {
      "url": "https://gateway.pipeworx.io/openremedies/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/openremedies/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1875+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/resolve_condition \
  -H 'Content-Type: application/json' \
  -d '{"q":"migraine","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/resolve_condition`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "openremedies": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-openremedies"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-openremedies
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Openremedies data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
