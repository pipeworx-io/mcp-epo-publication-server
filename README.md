# @pipeworx/epo-publication-server

As-published European patent documents — publication dates, per-date listings
with titles and applicants, and full document text (biblio, abstract, claims,
description) for granted patents and directly-filed applications. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1837+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `eps_publication_dates(year?, from?, to?, limit?)` — EPO publication dates
  (weekly, almost always Thursdays, back to 1978) with a real per-date
  document count.
- `eps_list_publications(date, kind?, limit?)` — every EP publication issued
  on one date, with kind code, title and applicant(s); reports the true total
  before any filtering.
- `eps_get_document(number, kind?, part?)` — one publication's bibliographic
  data, abstract, claims or description text, as published.

## Auth

Keyless. No account, no registration.

EPO states a fair-use cap of 5 GB per rolling 7-day window per calling IP,
confirmed live via the `x-quotaperweek-remaining`/`x-quotaperweek-used`
response headers. General EPO site terms apply; no EPS-specific
redistribution restriction was found.

## Data sources

- European Patent Office (EPO), European Publication Server — the
  as-published XML/HTML of every EP patent application and granted European
  patent.

## Notes for the next person

- **The list-by-date trap**: asking for a date that is not a real EPO
  publication date (wrong day of week, a future date, a typo) answers
  **HTTP 200** with a tiny HTML shell and zero entries — not a 404, not an
  error. EPO has never actually had a genuinely empty publication date in
  2,400+ weeks of history, so `eps_list_publications` treats zero parsed
  entries as a loud failure rather than a silent empty list. Always get a
  real date from `eps_publication_dates` first.
- **Most application (A-kind) publications carry NO claims/description text.**
  The large majority of EP applications enter via the PCT route, and EPO
  republishes only the bibliographic (SDOBI) record for those — the full
  specification was already published as the WIPO (WO) pamphlet, so EPO does
  not retype it. Full claims/description/drawings text is reliably present
  for **granted (B-kind)** patents and for **directly-filed** (non-PCT)
  applications. `eps_get_document` reports which case it hit (and names the
  referenced WO publication when it's a PCT case) rather than returning an
  empty part silently.
- **Document ids aren't just the publication number + kind.** The REST path
  segment is a longer id like `EP4819826NWA1` — the publication number, an
  internal marker (`NW` for the overwhelming majority of documents, `W1` for
  a minority), and the kind code. `eps_list_publications` returns these ids
  directly; `eps_get_document` also accepts a bare number + kind and tries
  the common id forms in order.
- **A made-up document id does not 404 — it 500s** with a small JSON body
  (`{"code":9000,"message":"Unhandled internal server error."}`). This is
  actually easy to work with: `eps_get_document` tries its candidate ids and
  reports `found:false` with the upstream status once all of them fail.
- What this adds over `mcps/epo-ops` (EPO Open Patent Services, OAuth2): the
  as-published document text itself, keyless. `epo-ops` keeps INPADOC
  worldwide legal status and CQL search across DOCDB (100+ patent offices);
  this pack only ever sees EP publications.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "epo-publication-server": {
      "url": "https://gateway.pipeworx.io/epo-publication-server/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/epo-publication-server/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1837+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/eps_publication_dates \
  -H 'Content-Type: application/json' \
  -d '{"year":"2026","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/eps_publication_dates`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "epo-publication-server": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-epo-publication-server"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-epo-publication-server
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Epo Publication Server data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
