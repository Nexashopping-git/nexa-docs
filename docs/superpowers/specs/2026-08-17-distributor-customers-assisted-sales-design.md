# Distributor Customers and Assisted Sales Design

## Scope

This release lets an authenticated distributor create or attach a customer by phone, maintain a private distributor-to-customer relationship, build a cart from the distributor's inventory, place a `DISTRIBUTOR_ASSISTED` order, and fulfil that sale. It prepares global acquisition attribution for later customer self-registration without building customer OTP login, pincode discovery, automatic seller ranking, referral rewards, payments, credit, or GST invoicing.

## Identity and attribution

An `Account` with role `CUSTOMER` remains the single global identity and its E.164 phone remains unique. A new `CustomerProfile` stores immutable acquisition data: `acquisitionSource` (`ORGANIC`, `REFERRAL`, or `DISTRIBUTOR_ASSISTED`), `acquiredByAccountId`, and `acquiredAt`. The first acquisition wins; attaching an existing customer to another distributor never overwrites it.

In this release, a new customer created by a distributor receives `DISTRIBUTOR_ASSISTED` attribution to that distributor. Existing customers retain their current attribution. Organic admin attribution and referral-code attribution are represented by the model but will be written by the future customer registration flow.

## Distributor relationship

`DistributorCustomer` is a many-to-many relationship with a unique `(distributorAccountId, customerAccountId)` key. It contains distributor-private `displayName`, `notes`, `status`, and timestamps. A distributor can see and edit only its own relationship. Global phone and account identity are returned only after the distributor has attached the customer.

`POST /customers` is idempotent by `(actor distributor, normalized phone)`: it creates the global customer when absent, otherwise reuses an existing `CUSTOMER`, and creates or returns the relationship. A phone belonging to a non-customer account returns `CONFLICT` and must not expose details about that account.

## Assisted sale

The distributor selects one of its customer relationships, browses its own inventory, and builds the customer's cart by passing `sellerAccountId = distributor.id` and `buyerAccountId = customer.id` to the existing cart API. Checkout calls `POST /orders` with `channel = DISTRIBUTOR_ASSISTED`, the same seller and buyer, and a shipping-address snapshot.

One cart and order always have one seller. Existing order reservation and fulfilment rules remain authoritative. The distributor views sales with `GET /orders?role=seller` and uses existing confirm, ship, deliver, and cancel transitions.

## Authorization and privacy

Only active distributors may use customer-management endpoints in this release. A distributor cannot list, read, or edit another distributor's relationship. Customer lookup is folded into idempotent attachment so there is no global phone-search endpoint. Assisted cart and order authorization continues to verify that the actor is the seller. Admin customer management requires an explicit distributor scope and is deferred rather than granting ambiguous global access.

## UI

The distributor panel gains Customers and Sales navigation items. Customers supports paginated search, creation/attachment, relationship details, and starting a sale. The assisted-sale flow selects listed, available inventory, manages a customer-scoped cart, captures an address, and places the order. Sales lists seller-side orders and exposes only state-machine-valid fulfilment actions.

## Verification

Server tests cover new-vs-existing customer attachment, immutable attribution, phone-role conflicts, relationship isolation, pagination, and assisted-order authorization. Panel verification covers lint, TypeScript/Next build, API error states, empty states, and a manual end-to-end path from customer attachment through delivered order and resulting inventory ledger entries.
