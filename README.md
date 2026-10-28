# @pipeworx/mn-drug-price-transparency

Minnesota manufacturer drug price-growth and new-drug-launch disclosures —
the Minnesota half of the fleet #3245 state WAC-transparency survey. See
`mcps/ca-drug-price-transparency/README.md` for the full per-state survey.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1877+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `mn_rx_price_growth(drug?, manufacturer?, ndc?, limit?)` — manufacturer
  disclosures of a reported WAC (list price) increase, filed under
  Minnesota's Prescription Drug Price Transparency program.
- `mn_rx_new_drug_launches(drug?, manufacturer?, limit?)` — manufacturer
  disclosures of a new drug launched above Minnesota's reporting threshold.

Both answer from a periodically-refreshed table (not a live query — Minnesota
publishes no API) and every response carries `data_as_of`.

## Auth

Keyless.

## Data sources

- Minnesota Department of Health (MDH), Prescription Drug Price
  Transparency program (Minn. Stat. 62J.84) — two periodic snapshot files:
  drugs with a reported WAC price increase, and new drugs launched above the
  reporting threshold. Not a per-year archive like California's CTRx program
  (see `mcps/ca-drug-price-transparency`), and no query API — refreshed by
  `scripts/ingest-mn-rx-transparency.mjs`.

## Traps

**MDH's export uses the literal text `"NULL"` as its missing-value
sentinel** (confirmed 2026-10-18 on `Patent Expiration Date` etc, each
paired with a same-row `... Not Applicable Indicator = 1`) — a real ingest
trap, since `"NULL"` is a non-empty, truthy string in JS and would otherwise
be stored as the four literal characters rather than `null`. The parser
checks for it explicitly.

**Some narrative free-text fields (e.g. `Price Increase Factors`) already
carry mojibake in MDH's own source file** — mis-decoded punctuation like
`doesnâ€™t` for "doesn't". This is MDH's export, not something this ingest
introduces; it is preserved as-is rather than guessed at, because a wrong
re-encoding guess is worse than a visibly broken apostrophe.

**No stable natural key.** The same NDC recurs across multiple price-growth
reports over time with no report id, so both tables use a surrogate
`bigserial` key and a full-snapshot replace rather than an upsert.

**A same-day rerun used to duplicate every row** (found 2026-10-19,
promoting this pack out of `mcps/_incubator/`): the delete-stale step ran
AFTER insert and only removed rows STRICTLY OLDER than the current
`data_as_of`, so dispatching the refresh workflow twice before MDH
republishes inserted a second full copy and deleted nothing. Fixed by
deleting everything at or before the current `data_as_of` BEFORE inserting
— see `scripts/ingest-mn-rx-transparency.mjs`'s header.

**MDH's CDN 403s a bare `curl`/`fetch` with no identifying User-Agent** — the
exact same URL with one set returns 200. Not an auth wall; checked in
`scripts/ingest-mn-rx-transparency.mjs`'s `UA` constant.

## Refresh

`.github/workflows/mn-rx-transparency-refresh.yml` runs monthly and on
`workflow_dispatch`: ingest, then install the pack's workspace deps, then
smoke the pack against the freshly loaded table with real credentials
(GitHub Actions secrets only — no agent machine holds the Supabase service
key).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "mn-drug-price-transparency": {
      "url": "https://gateway.pipeworx.io/mn-drug-price-transparency/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/mn-drug-price-transparency/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1877+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/mn_rx_price_growth \
  -H 'Content-Type: application/json' \
  -d '{"manufacturer":"Pfizer"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/mn_rx_price_growth`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "mn-drug-price-transparency": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-mn-drug-price-transparency"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-mn-drug-price-transparency
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Mn Drug Price Transparency data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
