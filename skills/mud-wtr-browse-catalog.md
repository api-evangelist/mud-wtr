---
name: mud-wtr-browse-catalog
description: Find MUD\WTR products and read their real prices, variants and availability without any credentials.
api: MUD\WTR Storefront MCP Server
generated: '2026-08-26'
method: generated
source: >-
  Grounded in mcp/mud-wtr-storefront-mcp-tools.json and graphql/mud-wtr-storefront.graphql, both
  captured live on 2026-08-26. Every tool name, GraphQL field and URL below appears in one of those
  artifacts.
operations:
  mcp:
    endpoint: https://mudwtr.com/api/mcp
    tools: [search_catalog, get_product_details, search_shop_policies_and_faqs]
  graphql:
    endpoint: https://mudwtr.com/api/2025-10/graphql.json
    fields: [products, product, productByHandle, search, predictiveSearch, collections, collectionByHandle]
  json:
    - GET https://mudwtr.com/products.json
    - GET https://mudwtr.com/collections.json
    - GET https://mudwtr.com/products/{handle}.json
---

# Browse the MUD\WTR catalog

No key, no token, no account. All three surfaces below answered anonymously when probed on
2026-08-26.

## 1. Search by intent (preferred)

POST `https://mudwtr.com/api/mcp` with a JSON-RPC 2.0 body:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"search_catalog","arguments":{"catalog":{"query":"rise"}}}}
```

The result content is a JSON string containing a `ucp` block plus a `products` array. Each product
carries a `gid://shopify/Product/<id>` identifier — that is the identifier every other tool wants.

## 2. Read one product in full

```json
{"jsonrpc":"2.0","id":2,"method":"tools/call",
 "params":{"name":"get_product_details",
           "arguments":{"product_id":"gid://shopify/Product/2296107466806",
                        "country":"US","language":"EN"}}}
```

Pass `options` to pin a specific variant. Without it you get the first available variant, which is
not necessarily the one the buyer meant — always echo the variant title back to the buyer before
adding it to a cart.

## 3. Or query GraphQL when you need shaped data

POST `https://mudwtr.com/api/2025-10/graphql.json`:

```graphql
{
  products(first: 10, query: "rise") {
    pageInfo { hasNextPage endCursor }
    edges { node { id title handle availableForSale
                   priceRange { minVariantPrice { amount currencyCode } }
                   variants(first: 10) { edges { node { id title availableForSale } } } } }
  }
}
```

Paginate with `first` + `after` from `pageInfo.endCursor`. The flat JSON alternative,
`GET /products.json?limit=50&page=2`, uses page-and-limit instead.

## 4. Answering policy questions

`search_shop_policies_and_faqs` exists on `/api/mcp` but returned an empty array for
"refund window" during the probe, so the index appears unpopulated. Fall back to the GraphQL `shop`
field (`refundPolicy`, `shippingPolicy`, `termsOfService`, `privacyPolicy`, `subscriptionPolicy`) or
to the published policy pages under `https://mudwtr.com/policies/`.

## Rules

- **Money is in minor units.** The tools return `{"amount": 2500, "currency": "USD"}` meaning $25.00.
  Divide by 100 for two-decimal currencies before quoting a price. This is stated in the tool
  descriptions themselves.
- **Errors arrive with HTTP 200.** Read `error` on JSON-RPC responses and `errors[]` on GraphQL
  responses; the status code will lie to you. See `errors/mud-wtr-problem-types.yml`.
- **Back off on 429.** The store's `agents.md` says the MCP endpoint is rate-limited per IP and
  publishes no numbers.
