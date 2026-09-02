`routers/shopify.py`

* `ORDERS_QUERY` / `ORDER_DETAIL_QUERY` — added `totalReceivedSet { shopMoney { amount currencyCode } }`
* `ORDERS_QUERY` / `ORDER_DETAIL_QUERY` — replaced `physicalLocation { legacyResourceId }` with `fulfillmentOrders(first: 1) { edges { node { assignedLocation { location { legacyResourceId } } } } }`
* `extract_order_fields_from_graphql()` — added `"total_received": money("totalReceivedSet")`
* `extract_order_fields_from_graphql()` — `location_id` now read via new `_fulfillment_location_id()` helper instead of `physicalLocation`
* Added `_fulfillment_location_id()` helper, shared by the extractor, `get_orders`, and `get_order_detail`
* `get_orders` — location filter switched to `_fulfillment_location_id()`, explicit `None` check
* `get_order_detail` — same swap for the ownership check


`tests/test_shopify_routes.py`

* `_graphql_order_node()` fixture — added `totalReceivedSet { shopMoney { amount currencyCode } }`; replaced `physicalLocation` with `fulfillmentOrders` (`edges[0].node.assignedLocation.location.legacyResourceId`)
* `test_get_orders_success` — added assertion `orders[0]["total_received"] == "1000.00"`
* `test_get_orders_filters_out_mismatched_location` — `mismatched` node override switched from `physicalLocation` to `fulfillmentOrders` pointing at `LOC2`
* Added `test_get_orders_drops_order_with_no_fulfillment_location` — regression test confirming an order with empty `fulfillmentOrders.edges` is silently excluded from the list, same behavior as the old null-`physicalLocation` case
* `test_get_order_detail_wrong_location_forbidden` — node override switched from `physicalLocation` to `fulfillmentOrders` pointing at `LOC2`
* `test_extract_order_fields_from_graphql_node` — added assertion `result["total_received"] == "1000.00"`
* Added `test_fulfillment_location_id_extracts_from_first_edge` — asserts `_fulfillment_location_id()` returns `"LOC1"` for a normal node
* Added `test_fulfillment_location_id_none_when_no_edges` — asserts `None` when `fulfillmentOrders.edges` is empty
* Added `test_fulfillment_location_id_none_when_field_missing` — asserts `None` when `fulfillmentOrders` key is absent entirely
