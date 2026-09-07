`routers/shopify.py`

* `ORDERS_QUERY` — replaced `fulfillmentOrders(first: 1) { edges { node { assignedLocation { location { legacyResourceId } } } } }` with `retailLocation { legacyResourceId }`
* `ORDER_DETAIL_QUERY` — same replacement
* Deleted `_fulfillment_location_id()` helper entirely — no longer used
* `extract_order_fields_from_graphql()` — `location_id` now reads `(order.get("retailLocation") or {}).get("legacyResourceId")` instead of `_fulfillment_location_id(order)`; removed temporary `retail_location_debug` debug key
* `get_orders` — location filter now uses `(loc_id := (o.get("retailLocation") or {}).get("legacyResourceId"))` instead of `_fulfillment_location_id(o)`
* `get_order_detail` — `order_location_id` now reads `(order.get("retailLocation") or {}).get("legacyResourceId")` instead of `_fulfillment_location_id(order)`



## `routers/shopify.py`

### 1. `ORDERS_QUERY`

**Before:**
```graphql
fulfillmentOrders(first: 1) {
  edges {
    node {
      assignedLocation {
        location { legacyResourceId }
      }
    }
  }
}
```

**After:**
```graphql
retailLocation { legacyResourceId }
```

---

### 2. `ORDER_DETAIL_QUERY`

Same replacement as #1 — swap the same `fulfillmentOrders(...)` block for `retailLocation { legacyResourceId }`.

---

### 3. `_fulfillment_location_id()` helper

**Delete entirely:**
```python
def _fulfillment_location_id(order: dict) -> Optional[str]:
    """
    Pulls the fulfillment location's legacyResourceId from the first
    fulfillmentOrders edge, replacing the deprecated physicalLocation
    field. An order can technically have multiple fulfillmentOrders across
    locations (split fulfillment); this app assumes one POS location per
    order, so only the first edge is used — same single-location
    assumption physicalLocation used to give us.
    """
    edges = (order.get("fulfillmentOrders") or {}).get("edges") or []
    if not edges:
        return None
    assigned_location = (edges[0].get("node") or {}).get("assignedLocation") or {}
    location = assigned_location.get("location") or {}
    return location.get("legacyResourceId")
```
No longer needed — nothing calls it after changes #4–#6 below.

---

### 4. `extract_order_fields_from_graphql()`

**Before:**
```python
"location_id": _fulfillment_location_id(order),
```
```python
"taxes_included": order.get("taxesIncluded"),
"retail_location_debug": (order.get("retailLocation") or {}).get("legacyResourceId"),
```

**After:**
```python
"location_id": (order.get("retailLocation") or {}).get("legacyResourceId"),
```
```python
"taxes_included": order.get("taxesIncluded"),
```
(the `retail_location_debug` line is removed — it was only for testing)

---

### 5. `get_orders` filter

**Before:**
```python
orders = [
    o
    for o in orders
    if (loc_id := _fulfillment_location_id(o)) is not None
    and str(loc_id) == location_id
]
```

**After:**
```python
orders = [
    o
    for o in orders
    if (loc_id := (o.get("retailLocation") or {}).get("legacyResourceId")) is not None
    and str(loc_id) == location_id
]
```

---

### 6. `get_order_detail`

**Before:**
```python
order_location_id = _fulfillment_location_id(order)
```

**After:**
```python
order_location_id = (order.get("retailLocation") or {}).get("legacyResourceId")
```

---

## Not required / unchanged

- `total_received` / `totalReceivedSet` — already done, untouched
- `taxes_included` / `taxesIncluded` — already done, untouched
- `extract_transaction_fields()` (REST webhook path) — out of scope, untouched
- `get_shop_and_store`, `verify_shop_owns_location`, `ensure_valid_token`, `shopify_graphql`, HMAC/SSE plumbing — untouched
- No `models.py` changes
- No frontend changes — `location_id` output key name is unchanged, frontend only reads that key, not how it's derived

---

## Commit message

```
fix(shopify): use retailLocation instead of fulfillmentOrders for location

fulfillmentOrders was a workaround for the deprecated physicalLocation
field, but it's semantically about fulfillment/shipping, not the retail
sale location. retailLocation is Shopify's actual documented replacement
for physicalLocation (confirmed via changelog: location field removed
from Order object, use retailLocation instead).

Verified against 30 real orders on Shopify-test-01 — retailLocation
populated correctly on 100%, no nulls.

Removes _fulfillment_location_id() helper, no longer needed.
```
