# @pipeworx/spp

SPP (Southwest Power Pool) real-time electricity data — generation mix, load and
renewable forecast vs actual, five-minute LMPs, and the raw Marketplace file
archive back to 2013, for the 14-state footprint from the Texas panhandle to
North Dakota.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `spp_generation_mix(limit?, detailed?)` — MW per fuel per 5-minute interval
  (coal, gas, nuclear, wind, solar, hydro, diesel fuel oil, energy storage, waste
  heat, waste disposal, other) plus total load. `detailed: true` splits each fuel
  into its Market and Self-scheduled halves over a rolling 365-day feed.
- `spp_load_forecast(limit?, include_future?)` — short- and mid-term load, wind
  and solar forecasts (STLF/MTLF, STWF/MTWF, STSF/MTSF) against actual load, wind
  and solar.
- `spp_lmp_latest(location?, limit?)` — the most recently published RTBM 5-minute
  interval, per settlement location (~1,600), with LMP split into MEC / MCC / MLC.
- `spp_files(dataset, path?, download?, limit?)` — browse or download the
  Marketplace file archive directly, for a specific historical interval or day.

## Auth

Keyless. No registration.

## Data sources

- <https://portal.spp.org/chart-api/gen-mix/asFile> — current-day generation mix CSV.
- <https://portal.spp.org/chart-api/gen-mix-365/asFile> — rolling 365-day, market
  vs self-scheduled.
- <https://portal.spp.org/chart-api/load-forecast/asFile> — load/wind/solar
  forecast vs actual CSV.
- <https://portal.spp.org/file-browser-api/?fsName=<dataset>&path=<path>&type=folder>
  — JSON directory listing of the Marketplace archive.
- <https://portal.spp.org/file-browser-api/download/<dataset>?path=<path>> — the file.

## Traps

- **`marketplace.spp.org` 302-redirects to `portal.spp.org`** for every API path.
  Call `portal.spp.org` directly so a Worker fetch never depends on redirect
  handling.
- **The load-forecast CSV is published a week ahead and is newest-first**, so the
  first ~2,000 rows are future intervals with every value blank. A naive
  `rows[0]` returns a row of `null`s as a clean success. Measured 2026-09-17:
  **2,017 forecast-only rows in front of 287 settled ones.** `spp_load_forecast`
  splits on "does this row have an Actual Load", reports `latest_actual` from the
  newest settled row, and exposes `forecast_only_intervals` so the caller can see
  the split. This is the pack's main reason for existing over a raw CSV fetch.
- **Two timezones in one dataset.** `gen-mix` timestamps are GMT; the LMP archive
  FILENAMES (`RTBM-LMP-SL-202609171905.csv`) are Central. Mixing them shifts you
  five or six hours. Inside the LMP file, `Interval` is Central and
  `GMTIntervalEnd` is UTC.
- **Don't compute the "current" LMP filename.** SPP publishes each 5-minute
  interval a few minutes late, so the filename for right now is usually a 404.
  `spp_lmp_latest` lists the day's `By_Interval` folder and takes the newest file
  that exists, falling back to the previous day just after midnight CT.
- **The file-listing endpoint needs no token.** SPP's own UI sends an
  `X-SPP-UI-Token` header; the server does not require it (verified 2026-09-17),
  so this pack sends none rather than forge one.
- **Blank means unknown, not zero.** `num()` returns `null` for an empty cell.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "spp": {
      "url": "https://gateway.pipeworx.io/spp/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/spp/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/spp_generation_mix \
  -H 'Content-Type: application/json' \
  -d '{"limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/spp_generation_mix`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "spp": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-spp"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-spp
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Spp data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
