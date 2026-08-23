---
name: johnnie-o-browse-catalog
description: Search and read the Johnnie-O apparel catalog anonymously, either through the
  store's UCP MCP tools or through its Storefront GraphQL endpoint.
api: johnnie-o:storefront-graphql
generated: '2026-08-23'
method: generated
source: graphql/johnnie-o-storefront.graphql + mcp/johnnie-o-mcp-tools-list.json
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - product
  - productByHandle
  - products
  - search
  - predictiveSearch
  - collection
  - collectionByHandle
  - collections
  - productRecommendations
---

# Browse the Johnnie-O catalog

Johnnie-O sells men's, women's and boys' apparel. Its catalog is readable **without any
credentials** on two different surfaces. Pick by what you need.

## Surface A — Storefront GraphQL (read anything in the schema)

```
POST https://www.johnnie-o.com/api/2024-10/graphql.json
Content-Type: application/json
```

No `Authorization` header. The storefront proxies the request and injects its own token.

Search:

```graphql
{ products(first: 10, query: "polo") { nodes { title handle
    priceRange { minVariantPrice { amount currencyCode } } } } }
```

One product by handle:

```graphql
{ productByHandle(handle: "boston-college-birdie-prep-formance-polo") {
    title description
    variants(first: 50) { nodes { id title availableForSale
      selectedOptions { name value }
      price { amount currencyCode } } } } }
```

Browse a collection, paginating with cursors:

```graphql
{ collectionByHandle(handle: "new-arrivals") {
    title
    products(first: 24, after: null) {
      nodes { title handle }
      pageInfo { hasNextPage endCursor } } } }
```

Pass `pageInfo.endCursor` back as `after:` for the next page. **Every** list on this API is a
GraphQL cursor connection — there is no page/offset parameter anywhere.

## Surface B — UCP MCP tools (agent-native)

```
POST https://checkout.johnnie-o.com/api/ucp/mcp
Content-Type: application/json
Accept: application/json, text/event-stream
```

`tools/list` is anonymous and returns 13 tools with full input schemas. For reading the
catalog use `search_catalog`, `lookup_catalog` and `get_product`.

**`tools/call` is not anonymous.** Without a UCP agent profile URI every call comes back as:

```json
{"jsonrpc":"2.0","id":1,"error":{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri"}}}
```

If you do not have an agent profile, use Surface A — it needs nothing.

## Rules that will bite you

- **GraphQL errors arrive with HTTP 200.** Check `errors[]` in the body, not the status code.
- **Prices in MCP responses are integers in ISO 4217 minor units** paired with a currency
  code: `{"amount": 2500, "currency": "USD"}` is $25.00. GraphQL prices are decimal strings
  in `MoneyV2`. The two surfaces do not agree on representation — do not mix them.
- **Meter yourself.** GraphQL returns `extensions.cost.requestedQueryCost` on every response;
  MCP returns `shopify-complexity-score` headers. The store's `llms.txt` says the MCP endpoint
  is rate-limited per IP and to back off on 429.
- **Pin your version.** The path segment is a calendar quarter. `2024-10` is what the store's
  own client uses; `2026-07` is the latest the API reports as supported. Query
  `{ publicApiVersions { handle supported } }` to see the live list before pinning.

See also: `conventions/johnnie-o-conventions.yml`, `errors/johnnie-o-problem-types.yml`,
`rate-limits/johnnie-o-rate-limits.yml`.
