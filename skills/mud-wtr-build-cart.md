---
name: mud-wtr-build-cart
description: Assemble and revise a MUD\WTR cart on the buyer's behalf, including discounts, gift cards and delivery details.
api: MUD\WTR Storefront MCP Server
generated: '2026-08-26'
method: generated
source: >-
  Grounded in mcp/mud-wtr-storefront-mcp-tools.json, mcp/mud-wtr-ucp-mcp-tools.json and
  graphql/mud-wtr-storefront.graphql, captured live 2026-08-26.
operations:
  mcp:
    endpoint: https://mudwtr.com/api/mcp
    tools: [get_cart, update_cart]
  graphql:
    endpoint: https://mudwtr.com/api/2025-10/graphql.json
    fields: [cart, cartCreate, cartLinesAdd, cartLinesUpdate, cartLinesRemove, cartBuyerIdentityUpdate,
      cartDeliveryAddressesAdd, cartDeliveryAddressesReplace, cartSelectedDeliveryOptionsUpdate,
      cartDiscountCodesUpdate, cartGiftCardCodesAdd, cartGiftCardCodesRemove, cartNoteUpdate,
      cartPrepareForCompletion]
---

# Build a MUD\WTR cart

## 1. Create the cart

There is no `create_cart` on the anonymous `/api/mcp` server — create it in GraphQL:

```graphql
mutation {
  cartCreate(input: {lines: [{merchandiseId: "gid://shopify/ProductVariant/<id>", quantity: 1}]}) {
    cart { id checkoutUrl totalQuantity cost { totalAmount { amount currencyCode } } }
    userErrors { code field message }
  }
}
```

Keep the returned `cart.id`. `cart.checkoutUrl` is the human hand-off — a real URL a buyer can open.

## 2. Revise it in one call

`update_cart` on `/api/mcp` collapses a dozen GraphQL mutations into a single tool call:

```json
{"jsonrpc":"2.0","id":3,"method":"tools/call",
 "params":{"name":"update_cart","arguments":{
   "cart_id":"gid://shopify/Cart/<id>",
   "add_items":[{"product_variant_id":"gid://shopify/ProductVariant/<id>","quantity":2}],
   "discount_codes":["SAMANTHAJO"],
   "note":"leave at side door"}}}
```

Available fields, verbatim from the tool's inputSchema: `add_items`, `update_items`,
`remove_line_ids`, `buyer_identity`, `delivery_addresses_to_add`, `delivery_addresses_to_replace`,
`selected_delivery_options`, `discount_codes`, `gift_card_codes`, `note`.

## 3. Read it back

```json
{"jsonrpc":"2.0","id":4,"method":"tools/call",
 "params":{"name":"get_cart","arguments":{"cart_id":"gid://shopify/Cart/<id>"}}}
```

Returns items, shipping options, discount info and the checkout URL.

## Undoing things

Cart state is fully reversible from the agent side, and cheaply:

| To undo | Use |
| --- | --- |
| a line you added | `update_cart` with `remove_line_ids`, or `cartLinesRemove` |
| a quantity change | `update_cart` with `update_items`, or `cartLinesUpdate` |
| a discount code | `cartDiscountCodesUpdate` with an empty list |
| a gift card | `cartGiftCardCodesRemove` |
| the whole cart | `cancel_cart` — **UCP server only**, `https://mudwtr.com/api/ucp/mcp`, and that server requires a Shopify agent JWT |

No window is published for any of these. See `conventions/mud-wtr-conventions.yml` → `reversibility`.

## Error handling

Cart mutations return typed `userErrors[]` with a `code` from `CartErrorCode` (55 values) — check
`errors/mud-wtr-problem-types.yml`. The ones that bite agents most are `MERCHANDISE_NOT_APPLICABLE`,
`MISSING_DISCOUNT_CODE`, `MINIMUM_NOT_MET`, and the whole `ADDRESS_FIELD_*` / `INVALID_ZIP_CODE_*`
family.
