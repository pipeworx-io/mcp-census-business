# @pipeworx/census-business

How big is an industry? U.S. Census Bureau business statistics by NAICS industry and geography: employer establishments, employees and payroll (County Business Patterns), nonemployer businesses, Economic Census receipts, employer-firm owner demographics (Annual Business Survey), and monthly retail and quarterly services sales series.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2034+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

Tools take a plain industry phrase ("dental offices") as well as a NAICS code, a state or county, and a year, and return values with the NAICS title, geography, year, Census dataset name and `data_as_of`.

## Tools

- `industry_code_search(query)` — industry words to the NAICS codes the Census business datasets use, from the Census Bureau's own industry list.
- `industry_size(industry, state?, county?, year?, breakdown?)` — establishments, employees, annual payroll and average pay for an industry in the nation, a state or a county (County Business Patterns), plus nonemployer establishments and receipts (Nonemployer Statistics). `breakdown: "state"` ranks every state; `"county"` ranks the counties of a state. Payroll and receipts are returned in dollars (Census publishes thousands).
- `establishment_size_split(industry, state?, county?, year?)` — establishments, employees and payroll by employment-size class (1-4, 5-9, ... 1,000+), with each class's share of establishments.
- `economic_census_industry(industry, state?, county?, year?)` — establishments, firms, sales / revenue, payroll and employees from the Economic Census (2022, 2017). The source for total industry revenue.
- `business_owner_demographics(industry?, dimension?, state?, year?)` — employer firms, receipts, employees and payroll by owner sex, ethnicity, race or veteran status (Annual Business Survey).
- `monthly_sales_series(program?, category_code?, data_type_code?, from?, to?, list_codes?)` — Monthly Retail Trade (mrts) and Quarterly Services (qss) series; `list_codes: true` shows the codes a program publishes.

For full NAICS descriptions, hierarchy, edition and SIC crosswalks, use the `naics` pack. For county employment and wages by industry from a different publisher (BLS), see `qcew`.

## Auth

Platform key (`PLATFORM_CENSUS_KEY`, the key the `census` pack also uses). The Census data service now answers 302 to a "missing key" page on every data query, so there is no keyless mode; callers do not pass a key.

## Data sources

- U.S. Census Bureau, County Business Patterns. Reference year 2023 is the latest; payroll is for the whole year, employment is for the week of March 12.
- U.S. Census Bureau, Nonemployer Statistics.
- U.S. Census Bureau, Economic Census (2022 and 2017 core statistics).
- U.S. Census Bureau, Annual Business Survey.
- U.S. Census Bureau, Monthly Retail Trade Survey and Quarterly Services Survey.

U.S. federal data, public domain; proxied live per request, nothing is copied.

## Things worth knowing

- **NAICS edition.** County Business Patterns still uses the 2017 NAICS through its 2023 data; the 2022 Economic Census and recent Annual Business Surveys use 2022. An industry renumbered in 2022 (software publishers: 511210 became 513210) has a different code in each. A code missing from a dataset's list is reported as such, with a pointer to `industry_code_search`; use the `naics` pack's `naics_crosswalk` to translate.
- **Nonemployer years.** The nonemployer tables moved to 2022 NAICS in 2022; `industry_size` pairs the CBP year with the latest nonemployer year on the same NAICS edition (2021) and says so rather than mixing editions.
- **Suppression.** Small cells are withheld or noise-infused. A missing row is reported as "published no ... row", never as zero; `employees_noise_flag` and `payroll_noise_flag` carry Census's own noise-range letter.
- **Firm-size tables.** Statistics of U.S. Businesses (firm size by industry) is bulk files only, not in the Census data service; `establishment_size_split` answers the size question by establishment employment class instead.
- **Verification status.** The data endpoints need a key no fleet machine holds, so this pack was built against the services' published metadata and a stubbed response shape; the first live answers come from the post-deploy probe.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "census-business": {
      "url": "https://gateway.pipeworx.io/census-business/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/census-business/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2034+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/industry_code_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"dental offices"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/industry_code_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-census-business` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/census-business/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "census-business": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-census-business"]
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
ask_pipeworx({ question: "your question about Census Business data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
