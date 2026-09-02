# TODO — Shopify `total_received` (use `totalReceivedSet` instead of `totalPriceSet`)

Related to Fuga-san's PR #83 review comment #1.
Backend first → verify → frontend consumes → verify → then open Linear issue for backend.

---

## 1. Backend — `taimatsu-tax-free-backend` (`routers/shopify.py`)

- [ ] `ORDERS_QUERY` — add `totalReceivedSet { shopMoney { amount currencyCode } }`
      (alongside existing `totalPriceSet` / `totalTaxSet` lines)
- [ ] `ORDER_DETAIL_QUERY` — same addition
- [ ] `extract_order_fields_from_graphql()` — add:
      ```python
      "total_received": money("totalReceivedSet"),
      ```
- [ ] Double-check `extract_transaction_fields()` (REST webhook path) — decide if
      `total_received` should also be added there for consistency, or left as a
      known gap (SSE push still uses `total_price`). Document the decision either
      way, don't leave it silently inconsistent.

### Backend testing

- [ ] **Unit test** — extend `extract_order_fields_from_graphql()` test coverage
      (find/add test file, likely `tests/routers/test_shopify.py` or similar):
      - Input: GraphQL order node with `totalPriceSet`, `totalTaxSet`,
        `totalReceivedSet` all populated with *different* values
      - Assert output dict's `total_received` matches `totalReceivedSet.shopMoney.amount`,
        and is distinct from `total_price`
- [ ] **Route test** — `GET /shopify/orders/{location_id}` and
      `GET /shopify/orders/{location_id}/{order_id}`:
      - Mock `shopify_graphql()` response to include `totalReceivedSet`
      - Assert response JSON includes `total_received` key
- [ ] **Manual check** — run backend locally (`docker compose up`), hit both
      endpoints against the Shopify test store (`Shopify-test-01`) directly
      (curl/Postman/Swagger UI), confirm `total_received` appears and is sane
      (matches `total_price` for a normal fully-paid POS order)
- [ ] Run full backend test suite (`pytest`) — confirm nothing else broke
      (e.g. any existing test asserting the exact shape of
      `extract_order_fields_from_graphql()`'s output dict, which will now have
      one more key)

---

## 2. Frontend — `taimatsu-tax-free-frontend` (only after backend is verified working)

- [ ] `ShopifyTransactionSelector.tsx` — `ShopifyOrder` interface: add
      `total_received: string;`
- [ ] `refillFromIds()` — change:
      ```js
      totalReceived += parseFloat(order.total_price || "0");
      ```
      to:
      ```js
      totalReceived += parseFloat(order.total_received || "0");
      ```
- [ ] Check other `total_price` usages in the same file — **do not blanket-rename**.
      The order-list row display (`formatJpy(totalAmount)`, ~line 598) and the
      detail dialog (`formatJpy(dialogDetail.total_price || "0")`, ~line 768) are
      showing order *value*, not payment *received* — decide deliberately whether
      those should also switch to `total_received` or intentionally stay as
      `total_price`. Note the decision in the PR description.

### Frontend testing

- [ ] File: `src/features/steps/shopify/shopify.test.tsx`
- [ ] Update mock order fixtures used across the file to include a
      `total_received` field (currently only `total_price`/`total_tax` are mocked)
- [ ] Extend/update existing test:
      **`"selects an order and fills order/tax/ids/received"`** (~line 227) —
      assert `onReceivedChange` is called with `total_received`'s value, not
      `total_price`'s
- [ ] Extend/update existing test:
      **`"selecting two orders merges totals, items, and IDs"`** (~line 269) —
      assert merged/summed received amount uses both orders' `total_received`
- [ ] Add a new test case: `total_received` differs from `total_price`
      (e.g. partial payment scenario) → assert the received field shown/passed
      to `onReceivedChange` reflects `total_received`, not `total_price`
- [ ] Run:
      - [ ] `npm run test` (Vitest) — confirm no regressions, new/updated tests pass
      - [ ] `npm run lint` (ESLint) — clean
      - [ ] `npm run build` (`tsc -b && vite build`) — confirm no TS errors from
            the new `total_received` field on `ShopifyOrder`
- [ ] Manual smoke test against staging: select a live Shopify order from the
      SSE list, confirm 支払合計 matches the real receipt

---

## 3. After both sides verified

- [ ] Open Linear issue on **backend** repo (`taimatsu-tax-free-backend`) —
      link back to SAM-401 / PR #83 as context
- [ ] Reference this checklist / decisions made (esp. the REST webhook gap and
      the "which `total_price` usages stay as-is" decision) in the issue or PR
      description so reviewers don't re-raise them
