---
name: Check a David Protein order and request a return
description: >-
  Read a signed-in customer's order status and store credit, and file a return, using the
  OAuth-protected customer-account MCP server.
api: mcp/david-protein-mcp.yml
server: https://account.davidprotein.com/customer/api/mcp
operations: [get_most_recent_order_status, get_order_status, get_store_credit_balances, request_return]
method: generated
generated: '2026-08-11'
grounded_in: mcp/david-protein-customer-account-mcp-tools.json
---

# Check a David Protein order and request a return

`POST https://account.davidprotein.com/customer/api/mcp`. Four tools, all scoped to the
customer the token represents — there is no customer id parameter anywhere.

## Authentication

OAuth 2.0 authorization code with PKCE (`S256`).

- Discovery: `https://davidprotein.com/.well-known/openid-configuration`
- Authorize: `https://account.davidprotein.com/authentication/oauth/authorize`
- Token: `https://account.davidprotein.com/authentication/oauth/token`
- Scope for this surface: `customer-account-mcp-api:full` (plus `openid email`)

That scope is all-or-nothing. There is no read-only order scope, so a token that can read
an order status can also file a return. Ask for it only when the buyer's request actually
needs it, and say so plainly when you do. See `scopes/david-protein-scopes.yml`.

## Steps

1. **"Where is my order?"** with no order number → `get_most_recent_order_status` (no
   arguments).
2. **A specific order** → `get_order_status` with `order_number` (the buyer-facing number,
   e.g. `#MX1001`).
3. **Store credit** → `get_store_credit_balances` (no arguments). Returns balances only if
   the customer has any.
4. **Return** → `request_return` with either `order_id` (a GID such as
   `gid://Shopify/Order/1234567890`) or `order_number`, plus `line_items` as
   `[{line_item_id, quantity}]`. Give return reasons in **human-readable form** — "size too
   large", not `SIZE_TOO_LARGE`. The tool description says so explicitly.

## Rules

- `request_return` is a write against a real order. Confirm the specific items and
  quantities with the buyer before calling it. There is no idempotency key on this tool, so
  do not retry a call whose outcome you did not read.
- Errors are JSON-RPC error objects under HTTP 200.
- Refund and return terms live at `https://davidprotein.com/policies/refund-policy`; quote
  those rather than inventing eligibility rules.
