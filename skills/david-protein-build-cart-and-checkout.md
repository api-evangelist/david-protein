---
name: Build a cart and take a David Protein order to checkout
description: >-
  Assemble a cart, open a checkout, set fulfillment, and stop at the human-approval
  boundary before payment, using the UCP commerce MCP server.
api: mcp/david-protein-mcp.yml
server: https://davidprotein.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product, create_cart, update_cart, get_cart, cancel_cart, create_checkout, update_checkout, get_checkout, complete_checkout, cancel_checkout]
method: generated
generated: '2026-08-11'
grounded_in: mcp/david-protein-ucp-mcp-tools.json
---

# Build a cart and take a David Protein order to checkout

`POST https://davidprotein.com/api/ucp/mcp`. Thirteen tools, Universal Commerce Protocol
version `2026-04-08`.

## Before the first call

- **Authentication.** `tools/list` is anonymous; every `tools/call` needs a signed JWT.
  Without one you get JSON-RPC `-32000` `AuthenticationRequired`.
- **Agent identity.** Every tool requires `meta.ucp-agent.profile` — a fetchable URI
  describing your agent. Omit it and the call fails with `-32001` `UCP discovery failed`,
  `data.code: invalid_profile_url`. This is not optional and it is checked before anything
  else.

## Steps

1. `search_catalog` or `lookup_catalog` to resolve what the buyer wants into product and
   variant GIDs. `get_product` returns one product with exact pricing and real-time
   availability, and supports interactive option selection through `selected`.
2. `create_cart` with the chosen line items. Hold the returned cart id.
3. `update_cart` to change quantities or lines; `get_cart` to re-read totals. **There is no
   idempotency key on cart mutations** — a retried `update_cart` can double-apply. Confirm
   with `get_cart` instead of blind-retrying.
4. `create_checkout` from the cart. The response carries line items, totals, discounts and
   taxes.
5. `update_checkout` to set the shipping address and shipping method. Fulfillment is
   single-destination only: the store's UCP profile declares
   `allows_multi_destination.shipping: false` and the only method combination is
   `[shipping]`.
6. **Stop.** Present the totals to the human.

## The payment boundary — read this before calling complete_checkout

`robots.txt` and `agents.md` both say the same thing: checkout, payment and order placement
must **not** be completed automatically. No scripted form fills, no browser automation, no
end-to-end flow that finalizes payment without an explicit, contemporaneous human approval
step. If you cannot get that approval at the moment of payment, David directs you to route
the purchase through the Shop skill (`https://shop.app/SKILL.md`) and Shop Pay instead.

When you do have approval:

7. `complete_checkout` with the checkout `id` **and** `meta.idempotency-key`. This is the
   only tool on the whole surface that accepts an idempotency key — use it on every attempt,
   and reuse the same key when retrying a call you are not sure landed. The response returns
   the order id and the thank-you page URL, or errors.
8. `cancel_checkout` / `cancel_cart` to abandon cleanly.

## Payment handlers available

`com.google.pay`, `dev.shopify.card` (visa, master, american_express, discover,
diners_club) and `dev.shopify.shop_pay`, per `well-known/david-protein-ucp.json`.
