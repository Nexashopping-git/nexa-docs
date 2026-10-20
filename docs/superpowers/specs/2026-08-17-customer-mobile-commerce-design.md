# Customer Mobile Commerce API Design

## Scope

The server will expose customer-only mobile APIs and a production-only OpenAPI 3.1 contract. No customer web or mobile UI is built in these repositories. Payments, credit, referral rewards, shipping-provider integration, and GST invoicing remain outside this release.

## Customer identity

A verified E.164 phone identifies one global `CUSTOMER` account. Customer OTP registration accepts an optional distributor referral code. A valid code creates immutable `REFERRAL` acquisition attribution and a `DistributorCustomer` relationship; no code creates immutable `ORGANIC` attribution to the configured primary admin. Customer accounts are created only after successful OTP verification.

## Catalogue and seller resolution

The mobile application receives one global catalogue. It never discovers or selects distributors. A referred customer's attributed distributor is the preferred fulfiller; an organic customer's fulfiller is admin.

For each complete cart, the server checks whether the attributed distributor has every listed variant and quantity. If yes, the distributor fulfils the entire order. If any line is unavailable, the server checks admin for the entire cart. If admin can fulfil it, the entire order moves to admin. Orders are never split. If neither can fulfil the complete cart, checkout reports the unavailable lines.

## Pricing

Admin stock retains `sellPrice` as the wholesale distributor price and gains `customerPrice` plus `minimumRetailPrice`. Distributor `sellPrice` and `discountPrice` must be between the primary admin's minimum retail price and customer price, and never exceed variant MRP. Discount price cannot exceed sell price.

The customer uses a dedicated seller-neutral `CustomerCart`. `POST /customer/cart/quote` resolves the fulfiller and snapshots line prices for a short validity period. If a distributor-backed quote falls back to admin during order placement, admin honours the quoted customer prices. Expired quotes must be refreshed; totals never increase silently.

## Orders

Customer order placement consumes a valid quote, reserves the resolved fulfiller's complete stock, and creates one existing `Order`. It snapshots `attributedDistributorAccountId`, `fallbackFromDistributorAccountId`, actual `sellerAccountId`, prices, and address. Existing seller fulfilment transitions remain authoritative.

## OpenAPI

`nexa-server/docs/openapi/customer-mobile.yaml` documents only customer mobile endpoints and includes only `https://nexashopping.duckdns.org/api/v1` in `servers`. Every operation includes authentication, request and response schemas, examples, pagination, and stable error responses. The specification is tracked with the deployed server, validated before handoff, and describes only deployed behavior.
