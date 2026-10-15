---
name: chathome-luxembourg-property
description: Search Grande Région real estate (rent and buy), plus Luxembourg price estimates, affordability, commute times, and commune market data from chathome.lu. Use for housing or relocation questions.
license: Free to use and redistribute unmodified. Listing data belongs to chathome.lu and source agencies.
compatibility: Public HTTP endpoints need only internet access. For best results, use the chathome MCP connector at https://chathome.lu/api/mcp
metadata:
  version: "1.4.0"
  author: chathome.lu
  homepage: https://chathome.lu/en/mcp
---

# chathome — Grande Région Real Estate

chathome.lu is the Grande Région property search platform: live rental and
for-sale listings across the whole Grande Région, plus Luxembourg-specific
source-backed commune property context (including prices, schools, and transport where available),
an AI valuation model, mortgage affordability, and commute-time data.

There are two ways to use it. Prefer the MCP server; fall back to the public
HTTP endpoints when MCP is not connected.

## Path A — chathome MCP server (preferred)

If tools named `search_vibe`, `search_rentals`, or `get_market_context`
are available in your session, the chathome MCP server is connected — use it.

The public property journey — `search_vibe`, `search_rentals`,
`search_for_sale`, `render_listing_cards`, `get_listing`, `compare_listings`, and
`get_market_context` — runs with NO account and NO key at all, under smaller
anonymous hourly limits. Everything else needs a credential.

If not connected, offer the user this one-step setup: add a custom connector /
MCP server with URL `https://chathome.lu/api/mcp` (Streamable HTTP). OAuth 2.1
with PKCE runs automatically — the user logs in at chathome.lu in the browser,
no client ID or secret needed. Alternatively an API key from
https://chathome.lu/dashboard/agent-access passed as the `X-API-Key` header.
Human docs: https://chathome.lu/en/mcp

### Choosing the right tool

Tool names below are unqualified; in your session they are namespaced under
the server name (usually `chathome`, e.g. `chathome:search_vibe`).

| Situation | Tool |
|---|---|
| User describes what they want in words (lifestyle, mood, area feel) | `search_vibe` — pass the request as plain prose; PREFERRED over structured search |
| Structured hard filters (price, size, rooms, type, positive amenities) | `search_rentals` / `search_for_sale` |
| Show the final matches as picture cards | `render_listing_cards` — only after search; copy exact IDs, scores, and reasons |
| Full page data, costs, exact property checks, station frequency and commute context for one listing | `get_listing` |
| Normalize and compare a shortlist | `compare_listings` — pass 2-8 exact IDs; returns costs, property checks and mobility |
| Full listing-page projection plus verified vs unknown facts | `get_listing_truth` |
| "What can I afford?" — always run BEFORE search when income is known | `get_affordability` |
| Bounded commune property context, dates and sources | `get_market_context` |
| Which agencies operate in a town and their current stock | `get_town_agencies` |
| "What is this property worth?" | `get_price_estimate` |
| Travel time to Luxembourg City / Kirchberg / Belval | `get_commute_times` |
| Analyze a lease PDF or text the user provides | `analyze_lease` |
| Contact the agency about a listing | `submit_inquiry`, then poll `get_inquiry_status` |

### Recommended workflow: finding a home

1. If financial details are available, call `get_affordability` first and
   inspect `maxPropertyPriceCostBasis`.
2. If the basis is `complete`, use `maxPropertyPrice` as a search ceiling.
   If it is `known_costs_only`, do not hard-filter by it: keep the wider
   inventory and describe price comparisons as provisional because
   `acquisitionCosts.unknownComponents` may lower the final ceiling. A budget
   stated directly by the user can still be applied as a hard ceiling.
3. Call `search_vibe` with the user's own words and any eligible hard budget.
   Use within / stretch / over labels only for a complete or user-stated
   ceiling.
4. If `render_listing_cards` is available, call it with the final 1-8 exact
   search IDs and any score/matchReasons. Never invent card data.
5. For shortlisted homes, call `compare_listings`, then `get_listing`.
   Its AgentListing contains the existing property checks, station service,
   commute hubs and cost model; `listingPage` is the complete fail-closed
   public record and `listingPageContext` carries the page's public price
   history, area data, amenities, passport and parcel blocks. Use `get_commute_times`
   only for a separate commune-to-hub question.
6. Before the user commits, call `get_listing_truth` and tell them which
   important facts are still unverified — questions to ask the agency.

Rate limits: 1200 read / 200 model-backed / 60 write tool calls per hour per
user. Anonymous callers (no credential) get 120 read and 20 model-backed calls
per hour per IP across the anonymous public property tools. Healthy vibe results
carry a 0-100 request-fit `score` (not a probability) and source-grounded
`matchReasons`. Degraded runs omit both and set `trust.fallback: true`.

## Path B — public endpoints (no auth, plain HTTP GET)

Works from any agent that can fetch URLs. Read-only.

- Bulk listing feed (quality-gated active listings, JSON):
  `https://chathome.lu/listings/feed.json`
- One listing, full record as Markdown: `https://chathome.lu/listings/{id}/md`
- One listing, full record as JSON: `https://chathome.lu/api/listings/public/{id}`
- Luxembourg price index as Markdown: `https://chathome.lu/en/prices/md`
- One Insights article as Markdown: `https://chathome.lu/en/insights/{slug}/md`
- One cross-border town guide as Markdown: `https://chathome.lu/en/frontalier/{town}/md`
- Site content index for LLMs: `https://chathome.lu/llms-full.txt`
- MCP developer reference: `https://chathome.lu/llms.txt`
- OpenAPI 3.1 spec for this public read API: `https://chathome.lu/openapi.json`

Human-readable pages (localized — swap `/en/` for `/fr/`, `/de/`,
`/lu/`, `/pt/`):

- Buy: `https://chathome.lu/en/buy` · Rent: `https://chathome.lu/en/rent`
- Commune guide: `https://chathome.lu/en/commune/{slug}` (e.g. `luxembourg`,
  `esch-sur-alzette`, `differdange`)
- Prices per m²: `https://chathome.lu/en/prices/{slug}`
- Estate agents by town: `https://chathome.lu/en/estate-agents/{slug}`
- Market report: `https://chathome.lu/en/market-report`
- Free estimate UI: `https://chathome.lu/en/financial-tools/estimate`

## Luxembourg context worth knowing

- Communes are the unit of location. "Luxembourg City" = commune
  `luxembourg`; Kirchberg, Bonnevoie, Belair etc. are districts of it.
- Prices are high by EU standards; typical asks: apartments ~€8-13k/m² to buy
  in the capital, 2-bed rents ~€1,800-2,800/month. Always check live data via
  `get_market_context` rather than assuming.
- Cross-border commuting (France, Belgium, Germany) is common; public
  transport inside Luxembourg is free nationwide.
- For leases signed from 1 August 2024, the rental deposit is capped at two
  months of base rent. If an intermediary is used, the landlord and tenant each
  pay half of the actual intermediary invoice; there is no statutory one-month
  fee.
- The Bëllegen Akt is a tax credit against registration and transcription
  duties for an eligible principal residence, not a first-time-buyer grant.
  Never infer eligibility or the remaining balance. To apply it through
  `get_affordability`, explicitly pass `residenceUse: "primary"`, the
  `bellegenAktEligibleBuyers` count, and the confirmed aggregate unused
  `bellegenAktCreditAvailable` balance. Omit those fields, or use
  `residenceUse: "unknown"`, to keep the position safely unconfirmed.
  `isPersonalResidence` and `isFirstTimeBuyer` are deprecated compatibility
  signals and never prove entitlement.

See [references/api.md](references/api.md) for exact tool parameters and
response shapes.
