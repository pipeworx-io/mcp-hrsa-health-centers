# @pipeworx/hrsa-health-centers

Where federally funded community health centers (FQHCs) and FQHC look-alikes operate: site address, phone, website, weekly hours, the operating health center, county, NPI and Medicare billing number, with distance search.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1905+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `hrsa_health_center_sites(name?, state?, city?, zip?, county?, near_zip?, near_city?, near_lat?, near_lon?, radius_miles?, program?, include_school_sites?, include_administrative?, limit?)` — finds health center sites. Answers "which FQHCs are in Miami", "community health centers within 10 miles of 33101", "is there an HRSA site called Riverside Community Health Center in Florida". With a center point (ZIP, city or coordinates) results sort by distance and carry `distance_miles`.

```
hrsa_health_center_sites({near_zip: "33101", radius_miles: 5, limit: 4})
  -> center: {basis: "...", approximate: true}, sites sorted by distance_miles
hrsa_health_center_sites({name: "Riverside", state: "FL"})
  -> found: false, reason: "no_match", plus near-name sites labelled as NOT matches
```

## Auth

Keyless.

## Data sources

- HRSA (Health Resources and Services Administration), Health Center Program site data: health center service delivery and look-alike sites. US federal data, public domain. The answer carries `data_as_of`, the publisher's record date, and `answered_from` (`stored_copy` or `live`).

The site list is one ~14 MB download (19.6k sites), too big to read per call, so a stored copy answers first; a loader refreshes it. Before the first load the pack reads the publisher's list directly (about 1.5 s on a cold call).

Caveats worth passing on:

- **Not-found is explicit.** A clinic missing here may be a free clinic, a rural health clinic, a hospital outpatient clinic, or listed under the health center's name rather than its own. The answer says so and shows similar names, labelled as non-matches. In the November 2026 data no Florida site is named "Riverside".
- **A quarter of the sites are school-based** (about 4,900 of 19,600). Searches without a name filter leave them out and report `school_sites_left_out`; pass `include_school_sites: true` to include them. Administrative-only offices are also left out unless `include_administrative` is true.
- **Distance centres.** `near_zip` is placed from the HRSA sites in that ZIP; when none exist, from the average of sites sharing the first three digits, and the answer says `approximate`. About 2% of sites carry no coordinates and are skipped in distance searches (counted in the answer). Pass `near_lat`/`near_lon` for an exact point.
- **Unknown state.** For about 375 sites HRSA lists the site state as unknown; the pack uses the operating health center's state and says so on the row.
- **Fields are positional in the stored shards** (see the header of `src/index.ts` and `scripts/lib/hrsa-sites-parse.mjs`); the test suite requires the loader and the live path to produce identical shards.
- Contact data is facility data only (site address, main phone, site website). No individual names are collected.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "hrsa-health-centers": {
      "url": "https://gateway.pipeworx.io/hrsa-health-centers/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/hrsa-health-centers/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1905+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/hrsa_health_center_sites \
  -H 'Content-Type: application/json' \
  -d '{"state":"FL","city":"Miami","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/hrsa_health_center_sites`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-hrsa-health-centers` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/hrsa-health-centers/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "hrsa-health-centers": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-hrsa-health-centers"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Hrsa Health Centers data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
