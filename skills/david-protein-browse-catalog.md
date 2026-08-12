---
name: Browse the David Protein catalog
description: >-
  Search David's products and read full product detail with no credentials, using the
  anonymous storefront MCP server.
api: mcp/david-protein-mcp.yml
server: https://davidprotein.com/api/mcp
operations: [search_catalog, get_product_details, search_shop_policies_and_faqs]
method: generated
generated: '2026-08-11'
grounded_in: mcp/david-protein-storefront-mcp-tools.json
---

# Browse the David Protein catalog

The storefront MCP server at `https://davidprotein.com/api/mcp` answered both `tools/list`
and `tools/call` anonymously when probed on 2026-08-11. Use it for anything read-only. Do not
reach for the UCP commerce server at `/api/ucp/mcp` to browse — its `tools/call` requires a
signed JWT.

## Transport

`POST https://davidprotein.com/api/mcp` with `Content-Type: application/json` and
`Accept: application/json, text/event-stream`. JSON-RPC 2.0.

## Steps

1. **Search.** Call `search_catalog` with `catalog.query` set to the buyer's words. Pass
   `catalog.context.address_country` and `catalog.context.currency` — `agents.md` states
   pricing and availability are wrong without them. Narrow with `catalog.filters.price.min`
   / `.max` (integer minor units) and `catalog.filters.available`.
2. **Page.** Read `pagination.cursor` from the response and send it back as
   `catalog.pagination.cursor` with `catalog.pagination.limit` (default 10, minimum 1).
   Only fetch more pages when the buyer asks for more.
3. **Detail.** Call `get_product_details` with the product `id` from the search result — a
   Shopify GID such as `gid://shopify/Product/8653621264551`. Pass `selected` to resolve a
   specific variant option.
4. **Policy questions.** Route "what's your return policy", "how long is shipping",
   "what are your hours" to `search_shop_policies_and_faqs` with a natural-language
   `query`. Do not answer these from the catalog or from memory.

## Rules

- Every price is an integer in the currency's ISO 4217 minor units paired with a currency
  code: `{"amount": 600, "currency": "USD"}` is $6.00. Never render the integer as dollars.
- Errors arrive as a JSON-RPC `error` object under **HTTP 200**. Check `body.error`, not the
  status code. See `errors/david-protein-problem-types.yml`.
- The endpoint reports `x-shopify-mcp-api-version: unstable`. Re-read `tools/list` rather
  than caching a tool contract.
- Back off on `429`. No rate-limit headers are returned, so a refusal is the only signal
  you get.
