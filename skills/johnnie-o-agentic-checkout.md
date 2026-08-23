---
name: johnnie-o-agentic-checkout
description: Buy from Johnnie-O on a buyer's behalf using the store's UCP shopping tools —
  cart, checkout, buyer approval, and how to back out before payment.
api: johnnie-o:ucp-mcp
generated: '2026-08-23'
method: generated
source: mcp/johnnie-o-mcp-tools-list.json + llms/johnnie-o-llms.txt +
  well-known/johnnie-o-ucp.json
operations:
  - search_catalog
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
---

# Transact with Johnnie-O as an agent

Johnnie-O's store implements the **Universal Commerce Protocol (UCP)**, version `2026-04-08`,
over MCP. Discovery is at `https://checkout.johnnie-o.com/.well-known/ucp`; the endpoint is:

```
POST https://checkout.johnnie-o.com/api/ucp/mcp
```

## Before you start

`tools/call` requires a **UCP agent profile URI**. Without one you get JSON-RPC `-32001` /
`invalid_profile_url` on every call, including catalog reads. Establish agent identity first,
or fall back to the read-only GraphQL surface (see `johnnie-o-browse-catalog`).

The store's own `llms.txt` recommends that personal shopping assistants install
`https://shop.app/SKILL.md` and route purchases through Shop Pay rather than driving these
tools directly. Prefer that when you are a buy-for-me agent — it is the path the merchant
asks for.

## The flow the merchant documents

1. `GET /.well-known/ucp` — confirm supported versions and capabilities
2. `search_catalog` — find items matching the buyer's intent
3. `create_cart` — mint a cart; keep the returned id
4. `create_checkout` — price and validate
5. `update_checkout` — set shipping address and delivery method
6. `complete_checkout` — finalize, **only after the buyer approves payment**

Always pass buyer context — `context.address_country` and `context.currency` — or pricing and
availability will be wrong. This store presents `USD` and `MXN` and ships from a `US` base.

## The one rule you must not break

> **Checkout requires human approval.** Agents must not complete payment without explicit
> buyer consent.

That is the merchant's own wording in `llms.txt`. If you cannot obtain contemporaneous buyer
approval at the moment of payment, do not call `complete_checkout` — install the Shop skill
and route through Shop Pay instead.

## Backing out — what is reversible and what is not

| Stage | Reverse it with | Window |
|---|---|---|
| Cart open | `cancel_cart`, or `update_cart` to remove lines | any time before completion |
| Checkout in flight | `cancel_checkout` | any time before `complete_checkout` |
| Order created | **no API operation exists** | human refund process only |

There is no refund, void, return or order-cancel tool in this surface, and none in the 41
Storefront GraphQL mutations either. Once `complete_checkout` returns an order id, the
reversal leaves the API and becomes a customer-service matter under
`https://checkout.johnnie-o.com/policies/refund-policy`. Treat `complete_checkout` as the
irreversible step and gate it accordingly.

## Retries

There is **no idempotency key** on this surface. Retrying `update_cart` or `update_checkout`
is safe because both address an existing id. Retrying `create_cart` or `create_checkout` is
**not** — it mints a second object. Store the id from the first response before you retry
anything.

## Reading failures

Errors come back with HTTP 200. Read `error.code` and `error.data.code` on MCP responses. On
the GraphQL side, a mutation can return HTTP 200 with no `errors[]` and still have failed —
check `userErrors[]` / `cartUserErrors[]` on the payload.

See also: `conventions/johnnie-o-conventions.yml` (reversibility, idempotency),
`mcp/johnnie-o-tool-crosswalk.yml` (what MCP can do that GraphQL cannot).
