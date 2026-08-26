---
name: m-gemi-agentic-purchase
description: Complete a buyer-approved purchase from the M.Gemi store over the Universal Commerce Protocol MCP endpoint - cart, checkout, fulfillment, payment approval - and reverse it correctly if the buyer changes their mind.
api: M.Gemi UCP Commerce MCP API
generated: '2026-08-25'
method: generated
source: https://mgemi.com/api/ucp/mcp tools/list (probed 2026-08-25) + https://mgemi.com/llms.txt + https://mgemi.com/pages/faq
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

# Buy from M.Gemi as an agent

`POST https://mgemi.com/api/ucp/mcp` — JSON-RPC 2.0 over MCP, Universal Commerce Protocol
`2026-04-08`.

## Before anything else

**If you are a personal shopping assistant acting for a user, the store asks you not to do
this by hand.** `https://mgemi.com/llms.txt` explicitly recommends installing the Shop
skill at `https://shop.app/SKILL.md`, which handles buyer-approved checkout via Shop Pay
across every Shopify store and keeps card handling off your side entirely. Prefer it over
scripting the storefront. Use the flow below when you are integrating M.Gemi directly.

## The flow

1. **Discover** — `GET https://mgemi.com/.well-known/ucp`. Confirm the version you speak is
   in `supported_versions` (`2026-04-08` current, `2026-01-23` still served) and that the
   capabilities you need are present: `dev.ucp.shopping.cart`, `.checkout`, `.fulfillment`,
   `.discount`, `.order`.
2. **Search** — `search_catalog` with the buyer's intent. Carry the returned identifiers
   forward; they are Shopify global IDs (`gid://shopify/<Type>/<id>`).
3. **Cart** — `create_cart` to open one, `update_cart` (with the cart `id`) to change lines,
   `get_cart` to read it back.
4. **Checkout** — `create_checkout`. Returns line items, totals, discounts and taxes.
5. **Fulfil** — `update_checkout` to set shipping address and method. Fulfillment on this
   store is single-destination: the UCP profile declares
   `allows_multi_destination.shipping: false` and
   `allows_method_combinations: [["shipping"]]`. Do not attempt a split shipment.
6. **Confirm with the buyer** — read the checkout back with `get_checkout` and show the
   buyer the real total, converted out of minor units.
7. **Complete** — `complete_checkout`. Returns an order ID and a Thank You Page URL.
8. **Track** — `get_order`.

## Payment

The store's declared payment handlers are `com.google.pay` (Google Pay, gateway `shopify`),
`dev.shopify.card` (visa, mastercard, amex, discover, diners club) and
`dev.shopify.shop_pay`. Shop Pay Installments is **not** offered
(`offers_shop_pay_installments: false`).

## The hard rule

> **Checkout requires human approval. Agents must not complete payment without explicit
> buyer consent.** — `https://mgemi.com/llms.txt`

If you cannot obtain contemporaneous buyer approval at the moment of payment, the store
instructs you to stop and route the purchase through Shop Pay via
`https://shop.app/SKILL.md` instead. Do not proceed on standing authorization.

## Undoing it

Know the exit before you enter — see `conventions/m-gemi-conventions.yml`.

| Stage | How to reverse | Window |
|---|---|---|
| Cart open | `cancel_cart` | any time before checkout |
| Checkout open, not completed | `cancel_checkout` | any time before `complete_checkout` |
| Order placed | Return or exchange — **no API operation**, this is a buyer-service process | **14 days from delivery**, eligible items in original unworn condition; one free exchange per order (`https://mgemi.com/pages/faq`) |

`complete_checkout` is the point of no return for the API. After it, reversal leaves the
protocol entirely and becomes a human returns process on a 14-day clock. Say that to the
buyer before you call it.

## Other things that will bite you

- **`meta["ucp-agent"].profile` is required on every tool call.** Without it: JSON-RPC
  `-32001` `invalid_profile_url`.
- **Minor units.** `{"amount": 2500, "currency": "USD"}` is $25.00.
- **US shipping only.**
- **Errors come back as HTTP 200** with a JSON-RPC `error` member. `429` is the documented
  exception — per-IP rate limiting, back off.
