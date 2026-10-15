# chathome API & MCP tool reference

MCP endpoint: https://chathome.lu/api/mcp (Streamable HTTP, MCP spec 2025-11-25)
Auth: OAuth 2.1 + PKCE (dynamic client registration) or `X-API-Key: cht_...`
Health / server info (no credential): https://chathome.lu/api/mcp/health
Discovery methods (initialize, ping, tools/list, prompts/list, resources/list)
need no credential. The public search, listing-card, listing-detail, and market
context tools also execute anonymously under hourly caps; every other
tools/call needs a credential.
Public REST spec (no credential): https://chathome.lu/openapi.json
Discovery: https://chathome.lu/.well-known/mcp.json · OAuth metadata:
https://chathome.lu/api/mcp/oauth/.well-known/oauth-authorization-server

## Tool parameters

### search_vibe (scope: listings:read)
- query (string, required): natural-language description, e.g. "quiet 2-bed
  near Kirchberg with a balcony under €2000"
- intent (string, optional): "rent" | "buy" (default "rent")
- limit (number, optional): 1-50, default 20
- locale (string, optional): en | fr | de | lu | pt ("lb" accepted as deprecated
  alias for "lu")
- includePhotos (boolean, optional, default true): attach thumbnail photos of
  the top results as MCP image content blocks

### search_rentals / search_for_sale (scope: listings:read)
- commune (string, optional): commune name or slug
- minPrice / maxPrice (number, optional): EUR (monthly rent, or sale price)
- minBeds (number, optional)
- minAreaSqm (number, optional)
- maxAreaSqm / minBathrooms (number, optional)
- propertyType (string, optional): apartment | house | studio | room
- furnished / petsAllowed / elevator / balcony / garden (optional true):
  require positive source evidence; unknown never counts as no
- parkingType (optional): off_street | garage | covered
- sort (optional): relevance | newest | price-asc | price-desc. Rental price
  sorting uses base rent, not an all-in monthly-cost claim.
- limit (number, optional): 1-50, default 20
- locale (string, optional)
- includePhotos (boolean, optional, default true)

### render_listing_cards (scope: listings:read)
- Call only after a property search.
- listings (array, required, 1-8): exact listing IDs in display order, with
  optional exact score and matchReasons copied from search_vibe.
- heading (string, optional): short heading grounded in the user's request.
- locale (string, optional): en | fr | de | lu | pt.
- Returns canonical listing facts again and links them to an interactive MCP
  Apps carousel. Clients without MCP Apps still receive structured data.

### get_listing (scope: listings:read)
- id (string, required) · locale (string, optional) ·
  includePhotos (boolean, optional, default true)
- Returns `listing` (normalized AgentListing v2), `listingPage` (the full
  fail-closed public projection), `listingPageContext` (price history, area
  trend, amenities, neighbourhood facts, passport, parcel and mobility), and trust metadata. The
  normalized record includes source and freshness, complete known/estimated
  rental costs, availability, property facts, official flood/noise screening,
  exact property checks, station frequency, commute hubs, value context and
  explicit unknowns.

### compare_listings (scope: listings:read)
- ids (array, required): 2-8 exact listing IDs from a search result
- locale (string, optional)
- Returns one `listing-comparison.v1` object containing normalized
  `AgentListing v2` records, explicit not-found IDs, and the fields intended
  for side-by-side comparison, including costs, property checks, transport and
  value context.

### get_listing_truth (scope: listings:read)
- id (string, required). Returns the complete fail-closed `listingPage` and
  `listingPageContext`, the normalized AgentListing decision record, known facts with sources and
  freshness, unknown-but-important facts, and a privacy-safe summary of
  user-reported corrections.

### get_market_context (scope: market:read)
- commune (string, required): slug, e.g. "luxembourg", "esch-sur-alzette"
- locale (string, optional)

### get_town_agencies (scope: market:read)
- commune (string, required): one of the supported town names or slugs
- locale (string, optional)
- Returns a dated active-listing count and buy/rent split, plus every agency's
  current town stock and asking-price ranges. Its figures match the localized
  `/estate-agents/{slug}` page because both read the same service.

### get_price_estimate (scope: market:read)
AI valuation model trained and backtested on historical Luxembourg listing
asking prices, not deed-level sales. Provide location, property type, area,
and specs; comparable statistics in the response are listing comparables.

### get_affordability (scope: market:read)
- Base inputs: `annualIncome`, `monthlyDebts`, `savingsAvailable`,
  `householdType`, `employmentType`, and `residencyType`.
- To apply Bëllegen Akt, all three facts must be explicit:
  `residenceUse: "primary"`; `bellegenAktEligibleBuyers` (confirmed
  eligible acquirers); and `bellegenAktCreditAvailable` (their confirmed
  aggregate unused balance). Omit confirmation fields or pass
  `residenceUse: "unknown"` to preserve the safe unconfirmed state; pass zero
  only when a zero count or balance has actually been confirmed.
- `isPersonalResidence` is an optional deprecated compatibility flag; prefer
  `residenceUse`.
- `isFirstTimeBuyer` is an optional deprecated compatibility signal (default
  false). It may
  support profile recommendations, but never confirms Bëllegen eligibility or
  balance.
- The response includes `maxPropertyPriceCostBasis` as `complete` or
  `known_costs_only`. Never use a known-cost-only figure as a hard search
  ceiling or a definitive affordability verdict; disclose
  `acquisitionCosts.unknownComponents`.
- Uses current Luxembourg mortgage rates and evidence-based acquisition costs.

### get_commute_times (scope: market:read)
- commune (string, required)
- destination: "luxembourg-city" | "kirchberg" | "belval"
- Four transport modes returned (car, transit, bike, walk).

### analyze_lease (scope: documents:write)
- Lease PDF as base64, or raw text. No URL input. Requires OAuth user auth.
  Returns a compact risk summary; full analysis is stored in the user's
  chathome account.

### submit_inquiry (scope: inquiries:write)
- listingId (string, required) · message (string, 10-5000 chars, required)
- contactEmail, agentName (optional)
- chathome-hosted listing → creates a conversation thread; external agency
  listing → returns agency contact details for a clean hand-off.

### get_inquiry_status (scope: inquiries:read)
- conversationId (string, required). Status: awaiting_reply | replied.

## Response shapes

Search tools return the exact returned count plus separately labelled match
counts. `matchingCountKind` says whether the count is exact, a lower bound,
or unavailable; `marketMatchingUpperBound` is never a final match count.
`nextCursor` is explicitly null until a stable cursor exists:

```typescript
type AgentListing = {
  schemaVersion: "agent-listing.v2";
  id: string; url: string; title: string;
  status: "active";
  price: number | null; currency: "EUR";
  listingType: "rent" | "sale"; propertyType: string | null;
  city: string | null; district: string | null;
  country: "LU" | "FR" | "BE" | "DE" | null;
  beds: number | null; baths: number | null; areaSqm: number | null;
  energyClass: string | null; imageUrl: string | null;
  photoUrls?: string[];            // up to 5, cover image first
  description?: string | null;     // short plain-text summary
  pricePerSqm?: number | null; charges?: number | null;
  features?: string[];             // amenity tags, e.g. "balcony"
  agencyName: string | null; latitude: number | null; longitude: number | null;
  source: {
    type: "original_agency_listing";
    agencyName: string | null; listingUrl: string;
    agencyListingId: string | null; lastSeenAt: string | null;
    status: "active" | "under_offer" | "unknown";
    statusBasis: "source_catalog_seen" | "listing_record_active";
    agencyConfirmedAt: string | null;
  } | null;
  rent: { baseMonthly: number | null; chargesMonthly: number | null;
    monthlyTotal: number | null; knownMonthlyLowerBound: number | null;
    deposit: number | null; agencyFee: number | null;
    knownUpfrontCost: number | null } | null;
  location: { city: string | null; district: string | null;
    precision: "exact_address" | "street" | "district" | "commune_only" | "unknown" };
  history: { firstObservedAt: string | null; priceDropTotalEur: number | null;
    lastChangedAt: string | null };
  duplicates: { confirmedOtherSources: number; confirmedSourceCount: number };
  amenities: Record<string, true | null>; // null = unknown, never an implied no
  fieldProvenance: Record<string, { publisher: string | null; method: string;
    sourceUrl: string | null; observedAt: string | null }>;
  missingFields: string[];
  match: { allRequiredConstraintsSatisfied: boolean | null;
    requiredConstraints: Array<{ id: string; status: string }>;
    reasons: string[] };
};
```

Search completeness fields are `returnedCount`, `matchingCount`,
`matchingCountKind`, `marketMatchingUpperBound`, `hasMore`,
`nextCursor`, and `searchExhaustive`.

Treat `source.listingUrl` as the primary-source link. `source.lastSeenAt` is the
latest agency-catalogue re-sighting, not proof that the detail page was fetched
then. A null `agencyConfirmedAt` means the agency has not directly confirmed
availability through ChatHome.

Healthy vibe results additionally carry request-fit `score` (0-100, not a
probability) and source-grounded `matchReasons` (up to 3 strings).

## Usage and attribution

Use public facts to answer the current user and link both the ChatHome page and
`source.listingUrl` when presenting a property. Do not infer bulk
redistribution, photo/description republication, caching, or model-training
rights from API availability. Respect rate limits and the published terms:
https://chathome.lu/terms#content-license

Unless `includePhotos: false`, search and detail responses append thumbnail
photos of the top results as MCP `image` content blocks (each preceded by a
text label naming the listing) — display them next to the matching listing
when presenting results to a human.

## MCP prompts (built-in)

- find-family-home — guided family-friendly rental search
  (args: commune?, budget?, beds?)
- match-properties — anonymous vibe search followed by interactive picture
  cards (args: criteria, intent?, locale?)

## Public no-auth endpoints

- https://chathome.lu/listings/feed.json — bulk feed of quality-gated active listings
- https://chathome.lu/listings/{id}/md — full listing record, Markdown
- https://chathome.lu/api/listings/public/{id} — full listing record, JSON
- https://chathome.lu/en/prices/md — Luxembourg price index, Markdown
- https://chathome.lu/en/insights/{slug}/md — localized Insights article, Markdown
- https://chathome.lu/en/frontalier/{town}/md — localized cross-border town guide, Markdown
- https://chathome.lu/openapi.json — OpenAPI 3.1 spec for the public read API
  (mirrored at https://chathome.lu/.well-known/openapi.json)
- https://chathome.lu/llms.txt — MCP developer reference
- https://chathome.lu/llms-full.txt — content index (listings, commune guides, blog)
- https://chathome.lu/.well-known/agent-card.json — A2A agent card
  (task endpoint https://chathome.lu/api/a2a)
