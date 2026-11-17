### Customer Identity and Geographic Consistency

The source `customers` table contains 99,441 records and 96,096 distinct `customer_unique_id` values.

- `customer_id` is unique across all source records and is associated with exactly one order.
- `customer_unique_id` represents the actual customer identity across multiple orders.
- 2,997 customers have placed more than one order.
- 250 customers have multiple ZIP code values.
- 122 customers have multiple city values.
- 39 customers have multiple state values.

**Data modeling implication:**

The proposed `DimCustomer` grain is one row per `customer_unique_id`. Geographic attributes should not be arbitrarily selected from one customer record. Instead, order-level geographic analysis should preserve the location associated with each order through its corresponding source customer record.

The observed geographic variations may reflect address changes or source-data inconsistencies. Their exact causes cannot be determined from the available data alone.


## 3. Order Items Grain and Duplicate Analysis

The `order_items` table contains 112,650 rows across 98,666 distinct orders.

### Key Uniqueness

- No duplicate combinations were found for `(order_id, order_item_id)`.
- The composite key `(order_id, order_item_id)` is suitable for identifying individual order items.
- The table grain is one row per order item within an order.

### Multiple Items per Order

- 9,803 orders contain multiple item rows.
- The maximum number of item rows per order is 21.
- 7,088 `(order_id, product_id)` combinations occur more than once.

### Repeated Product Analysis

Across the 7,088 repeated order-product combinations:

- 10,225 rows are additional occurrences beyond one row per combination.
- No combination contains multiple sellers.
- No combination contains multiple prices.
- No combination contains multiple freight values.
- No combination contains multiple shipping limit dates.

These patterns are consistent with multiple units of the same product being recorded as separate order-item rows. However, the source dataset does not provide an explicit quantity column, so this interpretation should remain documented as an observation rather than an independently verified business rule.

**Data modeling implication:**

The proposed `FactOrderItems` grain is one row per `(order_id, order_item_id)`. Repeated `product_id` values within an order are valid at this grain and must not be removed or aggregated during ingestion.

## 4. Orders Without Order Items

A total of 775 orders in the `orders` table have no corresponding records in `order_items`.

### Order Status Distribution

| Order Status | Count | Percentage |
|---|---:|---:|
| unavailable | 603 | 77.81% |
| canceled | 164 | 21.16% |
| created | 5 | 0.65% |
| invoiced | 2 | 0.26% |
| shipped | 1 | 0.13% |
| **Total** | **775** | **100%** |

### Assessment

- 767 orders (98.97%) have either `unavailable` or `canceled` status. Their lack of order-item records is broadly consistent with their order lifecycle status.
- Five `created` orders and two `invoiced` orders may represent incomplete order processing.
- One `shipped` order has no corresponding order-item record and requires further investigation.

**Data modeling implication:**

Orders without order-item records should not be automatically deleted. They should remain in the order-level dataset, while item-level metrics should be calculated exclusively from valid order-item records. The shipped-order exception should be investigated and documented separately.

### 4.1 Investigation of the Shipped Order Without Items

One exceptional order was identified among the 775 orders without corresponding order-item records.

**Order ID:** `a68ce1686d536ca72bd2dadc4b8671e5`

| Attribute | Value |
|---|---|
| Order Status | shipped |
| Purchase Timestamp | 2016-10-05 01:47:40 |
| Approval Timestamp | 2016-10-07 03:11:22 |
| Carrier Handover Timestamp | 2016-11-07 16:37:37 |
| Customer Delivery Timestamp | NULL |
| Payment Records | 1 |
| Payment Type | boleto |
| Payment Value | 77.73 |
| Review Records | 1 |
| Review Score | 1 |

**Assessment:**

The order has a recorded payment and customer review, despite having no corresponding order-item records. Its status is `shipped`, and a carrier handover timestamp is available, but the customer delivery timestamp is missing.

The review contains negative customer feedback, which is consistent with a potentially problematic purchase experience. However, the available data does not establish the exact cause of the missing order items or confirm successful delivery.

**Data modeling decision:**

- Retain the order in `FactOrders`.
- Retain its payment and review records in their respective fact tables.
- Do not fabricate missing order-item records.
- Exclude this order from item-based sales calculations unless the missing item data can be recovered from an authoritative source.
- Flag the order as an unresolved source-data quality exception.

## 7. Payment Data Quality and Reconciliation Analysis

### 7.1 Payment Data Grain and Cardinality

The `order_payments` dataset contains 103,886 records associated with 99,440 distinct orders.

Key findings:

- 2,961 orders have multiple payment records.
- The maximum number of payment records for a single order is 29.
- The composite key `(order_id, payment_sequential)` has no duplicate records.
- The proposed grain for `FactPayments` is one row per source payment record, identified by `(order_id, payment_sequential)`.

Payment method distribution:

- Credit card: 76,795 records (73.9%)
- Boleto: 19,784 records (19.0%)
- Voucher: 5,775 records (5.6%)
- Debit card: 1,529 records (1.5%)
- Not defined: 3 records

Multiple payment methods are observed within the same order. The most common combination is credit card and voucher, occurring in 2,245 orders.

**Modeling decision:** Preserve the original payment transaction grain. Do not aggregate or discard payment records during ingestion.

### 7.2 Payment-to-Order Value Reconciliation

Payment totals were aggregated by `order_id` and compared against aggregated order item values.

Definitions:

- Item Sales Value = SUM(order_items.price)
- Freight Value = SUM(order_items.freight_value)
- Order Item Value = Item Sales Value + Freight Value
- Payment Total = SUM(order_payments.payment_value)
- Payment Difference = Payment Total - Order Item Value

The comparison identified:

| Reconciliation Result | Orders |
|---|---:|
| Orders with both payment and item records | 98,665 |
| Payment records without corresponding item records | 775 |
| Item records without corresponding payment records | 1 |
| Matched orders with difference within 0.01 | 98,284 |
| Matched orders with difference greater than 0.01 | 381 |

Approximately 99.61% of matched orders reconcile within the 0.01 tolerance.

Among the 381 discrepant orders:

- 291 have payment totals greater than order item value.
- 90 have payment totals lower than order item value.
- 335 discrepant orders involve credit card payments.

### 7.3 Investigation of Payment Discrepancies

Further analysis was performed to investigate potential relationships between discrepancies, payment methods, installment counts, and order values.

Key observations:

- The Pearson correlation between installment count and absolute payment difference across all matched orders is approximately 0.068, indicating a very weak linear relationship.
- Credit card payments account for approximately 87.9% of discrepant orders.
- Two recurring positive discrepancy clusters were identified:
  - Approximately 13.02%–13.03% above order item value: 46 orders.
  - Approximately 15.49%–15.51% above order item value: 37 orders.
- The first cluster is concentrated mainly among 10-installment payments.
- The second cluster is more frequently associated with 12-installment payments.
- These clusters consist almost entirely of single-record credit card payments.
- The recurring percentage patterns appear across multiple months rather than being limited to a single period.

These observations suggest recurring patterns in credit card payment discrepancies. However, the available dataset does not provide sufficient information about bank settlement rules, installment financing, fees, or payment gateway calculations to establish their cause.

Therefore, these cases are classified as **unexplained payment reconciliation differences**, not confirmed payment-processing errors or bank fees.

### 7.4 Data Modeling and Processing Decisions

1. Preserve all original payment records in `FactPayments`.
2. Use `(order_id, payment_sequential)` as the candidate composite key.
3. Aggregate payment records only when performing order-level reconciliation or order-level reporting.
4. Aggregate payment and order item data independently before joining them to avoid join fan-out.
5. Preserve the distinction between:
   - Item sales revenue (`price`)
   - Freight value (`freight_value`)
   - Order item value (`price + freight_value`)
   - Collected payment amount (`payment_value`)
6. Do not fabricate missing item or payment records.
7. Retain discrepant orders and expose reconciliation status and difference as analytical quality indicators.
8. Investigate unexplained differences further only when additional authoritative payment or settlement information becomes available.

**Conclusion:** The payment dataset is suitable for loading into the proposed analytical warehouse at its original transaction grain. Most matched orders reconcile successfully, while a small subset contains unexplained differences that should remain visible for downstream monitoring and analysis.