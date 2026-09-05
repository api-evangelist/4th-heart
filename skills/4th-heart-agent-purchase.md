---
name: Buy from the 4th & Heart store as an agent
description: >-
  Search the 4th & Heart catalog, build a cart, run a checkout and complete a purchase over the
  store's anonymous Universal Commerce Protocol MCP endpoint, with the buyer approving payment.
api: mcp/4th-heart-mcp.yml
endpoint: https://fourthandheart.com/api/ucp/mcp
operations:
  - search_catalog
  - get_product
  - lookup_catalog
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-09-05'
method: generated
source: >-
  Tool names and required parameters verified against a live tools/list probe of
  https://fourthandheart.com/api/ucp/mcp on 2026-09-05, saved verbatim at
  mcp/4th-heart-mcp-tools.json. Rules are from https://fourthandheart.com/llms.txt.
---

# Buying from 4th & Heart as an agent

4th & Heart is a ghee and clarified-butter brand selling from a Shopify storefront. The store exposes
a Universal Commerce Protocol (UCP) surface over MCP, so you can transact without scraping the site.

## Connect

POST JSON-RPC 2.0 to `https://fourthandheart.com/api/ucp/mcp` with
`Content-Type: application/json` and `Accept: application/json, text/event-stream`.

No API key, token or account is required. `tools/list` and `initialize` answer anonymously.

## The one thing that will break your first call

Every `tools/call` requires a `meta` object identifying you:

```json
{ "meta": { "ucp-agent": { "profile": "https://your-agent.example/profile" } } }
```

The `profile` must be a resolvable URI — the server fetches it. Omit it and you get:

```json
{"error":{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri"}}}
```

with HTTP **422**. Read the body, not the status line: a protocol failure is a 422 carrying a
JSON-RPC error object.

## Flow

1. **Discover** — `GET https://fourthandheart.com/.well-known/ucp` to confirm the version and
   capabilities. Current is `2026-08-25`.
2. **Search** — `search_catalog` with `{meta, catalog}`. Use `get_product` for full detail on one
   item, or `lookup_catalog` to resolve several products or variants by identifier at once.
3. **Cart** — `create_cart` with `{meta, cart}`, then `update_cart` with `{meta, cart, id}` to
   change quantities. `get_cart` reads it back.
4. **Checkout** — `create_checkout` with `{meta, checkout}`.
5. **Fulfil** — `update_checkout` with `{meta, checkout, id}` to set shipping address and method.
6. **Complete** — `complete_checkout` with `{meta, checkout, id}`.
7. **Confirm** — `get_order` with `{meta, id}`.

## Rules you must follow

- **The buyer approves the payment, not you.** The store's published instructions require
  contemporaneous buyer consent at the moment of payment. If you cannot get it, do not call
  `complete_checkout` — route the purchase through Shop Pay instead.
- **Prices are integers in minor units.** `{"amount": 600, "currency": "USD"}` is $6.00. Divide by
  100 for two-decimal currencies before you quote a price to a person. JPY and other zero-decimal
  currencies are already whole units.
- **Pass buyer context.** Send `context.address_country` and `context.currency` so pricing and
  availability are correct.
- **Back off on 429.** The endpoint is rate limited per IP. No RateLimit headers are returned on
  success, so you will not see a limit coming — treat 429 as your only signal and retry with
  exponential backoff.

## Reversing what you did

- Before completion: `cancel_cart` and `cancel_checkout` exist. **No time window is published** for
  either, so cancel promptly rather than assuming you have time.
- After completion: **there is no refund, void or reverse tool.** A completed order can only be
  unwound out of band, by emailing `wecare@4thandheart.com` within the store's stated 30-day return
  window (https://fourthandheart.com/policies/refund-policy).
- **There is no idempotency key.** None of the 13 tools accepts one. A retried `complete_checkout` is
  not protected by contract — never blind-retry a completion. Confirm with `get_checkout` or
  `get_order` first.

## Reading the store without transacting

If you only need catalog data, these need no MCP call and no credential:

- `GET /products/{handle}.json` — one product
- `GET /collections/{handle}/products.json` — a collection
- `GET /search?q={query}&type=product` — search
- `GET /sitemap.xml` — everything
