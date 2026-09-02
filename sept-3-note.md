`routers/shopify.py`

* `ORDERS_QUERY` / `ORDER_DETAIL_QUERY` — added `totalReceivedSet { shopMoney { amount currencyCode } }`
* `ORDERS_QUERY` / `ORDER_DETAIL_QUERY` — replaced `physicalLocation { legacyResourceId }` with `fulfillmentOrders(first: 1) { edges { node { assignedLocation { location { legacyResourceId } } } } }`
* `extract_order_fields_from_graphql()` — added `"total_received": money("totalReceivedSet")`
* `extract_order_fields_from_graphql()` — `location_id` now read via new `_fulfillment_location_id()` helper instead of `physicalLocation`
* Added `_fulfillment_location_id()` helper, shared by the extractor, `get_orders`, and `get_order_detail`
* `get_orders` — location filter switched to `_fulfillment_location_id()`, explicit `None` check
* `get_order_detail` — same swap for the ownership check
