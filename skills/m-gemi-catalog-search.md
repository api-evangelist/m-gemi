---
name: m-gemi-catalog-search
description: Search and read the M.Gemi Italian footwear and leather-goods catalog as an agent, either through the store's UCP commerce MCP endpoint or through its documented read-only storefront JSON endpoints.
api: M.Gemi UCP Commerce MCP API
generated: '2026-08-25'
method: generated
source: https://mgemi.com/api/ucp/mcp tools/list (probed 2026-08-25) + https://mgemi.com/llms.txt
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Read the M.Gemi catalog

M.Gemi is a direct-to-consumer Italian footwear and leather-goods brand. Its store
publishes 2,407 products across 709 collections (`GET https://mgemi.com/meta.json`).
There are two ways to read it, and which you use depends on whether you intend to
transact.

## Path A — read-only, no profile needed

The store's own agent instructions (`https://mgemi.com/llms.txt`) document these as
available to agents that only need to read store data:

| Purpose | Request |
|---|---|
| All products | `GET https://mgemi.com/products.json` |
| One product | `GET https://mgemi.com/products/{handle}.json` |
| Collection products | `GET https://mgemi.com/collections/{handle}/products.json` |
| Search | `GET https://mgemi.com/search?q={query}&type=product` |
| Store metadata | `GET https://mgemi.com/meta.json` |
| Sitemap | `GET https://mgemi.com/sitemap.xml` |

No authentication. Prefer this path for browsing, comparison and answering questions —
it is cheaper and does not require a UCP agent profile.

## Path B — the UCP commerce MCP endpoint

`POST https://mgemi.com/api/ucp/mcp`, `Content-Type: application/json`,
`Accept: application/json, text/event-stream`. JSON-RPC 2.0.

Use this path when the read is a step toward a purchase, so the product and variant
identifiers you carry forward are the same ones cart and checkout expect.

Tools, verbatim from the live `tools/list`:

- **`search_catalog`** — search for products from the online store. Required: `meta`, `catalog`.
- **`lookup_catalog`** — look up multiple products or variants by identifier. Required: `meta`, `catalog`.
- **`get_product`** — retrieve complete product detail by identifier; returns a single product. Required: `meta`, `catalog`.

## Rules that will bite you

1. **Every tool call requires `meta["ucp-agent"].profile`** — a URI pointing at your UCP
   agent profile. `tools/list` and `initialize` work without it; a `tools/call` without it
   returns JSON-RPC error `-32001` `UCP discovery failed` / `invalid_profile_url`.
2. **Prices are integers in ISO 4217 minor units** paired with a currency code.
   `{"amount": 2500, "currency": "USD"}` is **$25.00**. Divide by 100 before quoting a
   price to a buyer for two-decimal currencies; JPY and other zero-decimal currencies are
   already whole units. This is stated in every tool description and is the single easiest
   way to be off by 100x.
3. **Pass buyer context** — `context.address_country` and `context.currency` — for accurate
   pricing and availability.
4. **The store ships to the US only** (`ships_to_countries: ["US"]`, currency USD). Do not
   promise delivery elsewhere.
5. **Errors arrive with HTTP 200.** Read the JSON-RPC `error` member, not the status line.
   The exception is `429`, which the store documents as a real per-IP rate limit — back off.

See also: `conventions/m-gemi-conventions.yml`, `errors/m-gemi-problem-types.yml`,
`rate-limits/m-gemi-rate-limits.yml`.
