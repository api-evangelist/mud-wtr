---
name: mud-wtr-agent-checkout
description: Run the UCP checkout flow against MUD\WTR — with the buyer-approval and idempotency rules the store actually requires.
api: MUD\WTR UCP Commerce MCP Server
generated: '2026-08-26'
method: generated
source: >-
  Grounded in mcp/mud-wtr-ucp-mcp-tools.json (live tools/list, 2026-08-26),
  https://mudwtr.com/.well-known/ucp, https://mudwtr.com/robots.txt, https://mudwtr.com/agents.md and
  https://mudwtr.com/policies/refund-policy.
operations:
  mcp:
    endpoint: https://mudwtr.com/api/ucp/mcp
    tools: [search_catalog, lookup_catalog, get_product, create_cart, get_cart, update_cart,
      cancel_cart, create_checkout, get_checkout, update_checkout, complete_checkout,
      cancel_checkout, get_order]
---

# Check out on MUD\WTR as an agent

**Read this section before writing any code.** The store's own `robots.txt` and `agents.md` state
the rule in identical language: *"Checkouts are for humans. Do NOT complete checkout, payment, or
order placement automatically — no scripted form fills, browser automation, or end-to-end agent
flows that finalize payment without an explicit, contemporaneous human approval step."* If you
cannot obtain that approval at the moment of payment, do not run step 5. Hand the buyer the cart's
`checkoutUrl` instead, or use the Shop skill the store recommends
(`https://shop.app/SKILL.md`).

## Before you start: this server is gated

`tools/list` on `https://mudwtr.com/api/ucp/mcp` is anonymous, but `tools/call` is not. A probe on
2026-08-26 returned:

```
-32000 AuthenticationRequired — "Unauthorized: A valid JWT is required to call <tool>."
```

and, separately, `-32001 UCP discovery failed / invalid_profile_url` when `meta.ucp-agent.profile`
was omitted. So **every** call needs both:

```json
"meta": { "ucp-agent": { "profile": "https://your-agent.example/ucp-profile" } }
```

plus a Shopify agent JWT as a bearer token
(`https://shopify.dev/docs/agents/get-started/authentication`).

## 1. Confirm capabilities

`GET https://mudwtr.com/.well-known/ucp`. It declares UCP `2026-04-08` (and `2026-01-23`), the
`dev.ucp.shopping` service on `transport: mcp`, and the capabilities in play here:
`dev.ucp.shopping.cart`, `.checkout`, `.fulfillment`, `.discount`, `.order`, `.catalog.search`,
`.catalog.lookup`. Fulfillment config says `allows_multi_destination.shipping: false` — one
shipping destination per order.

## 2. Find the merchandise

`search_catalog` → `lookup_catalog` / `get_product` to resolve exact variants.

## 3. Cart

`create_cart`, then `update_cart` to adjust. `cancel_cart` reverses the whole thing.

## 4. Checkout

`create_checkout`, then `update_checkout` to set shipping address and method. `get_checkout` reads
back line items, totals, discounts and taxes.

## 5. Complete — the point of no return

```json
{"jsonrpc":"2.0","id":9,"method":"tools/call",
 "params":{"name":"complete_checkout","arguments":{
   "id":"gid://shopify/Checkout/<id>",
   "meta":{"ucp-agent":{"profile":"https://your-agent.example/ucp-profile"},
           "idempotency-key":"<uuid you generate and keep>"},
   "checkout":{"payment":{ }}}}}
```

`meta.idempotency-key` is **required** by the tool's own schema — you cannot complete a checkout
without one. Generate it once per buyer intent and reuse it on every retry; never regenerate it
after a timeout, or you will risk a second order.

Payment failures come back as `CompletionErrorCode`. Retry only `PAYMENT_TRANSIENT_ERROR` and
`INVENTORY_RESERVATION_ERROR`. Never retry `PAYMENT_CARD_DECLINED`, `PAYMENT_INSUFFICIENT_FUNDS`,
`PAYMENT_INVALID_CREDIT_CARD` or `PAYMENT_CALL_ISSUER` — tell the buyer instead.

## 6. If it needs to be undone

- **Before completion:** `cancel_checkout` (checkout id) or `cancel_cart` (cart id). Both are
  callable; no window is published.
- **After completion:** there is no refund, void or reverse operation on any published MUD\WTR
  surface. The store's refund policy states refunds are unavailable "for ANY REASON on orders placed
  more than 30 days from original purchase date", and open tins are not refundable at all. The only
  route is a human emailing `drink@mudwtr.com` inside that 30-day window.

Tell the buyer this *before* step 5, not after.

## Prices

Integers in ISO 4217 minor units paired with a currency code: `{"amount": 2500, "currency": "USD"}`
is $25.00. The store's `paymentSettings` report USD / US.
