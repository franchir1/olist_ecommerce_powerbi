# Olist E-commerce — Analysis Layer

## Primary Key Validation and Structural Checks

This document describes the **primary key (PK) validation measures** implemented in the ANALYSIS layer.

These measures are **structural assertions**, not analytical KPIs.
Each measure returns the **number of PK violations** detected in the current model state.

**Expected value:** `0`
**Observed value > 0:** structural inconsistency that must be explained before analysis proceeds.

---

## Purpose of PK validation in the Analysis layer

The ANALYSIS layer is responsible for ensuring that the data model is:

- structurally coherent
- consistent with declared grain
- safe for relational joins

---

## Implemented PK validation measures

### dim_customer — PK check

- **Logical PK:** `customer_unique_id`
- **Measure name:** `Analysis - dim_customer PK`

```DAX
Analysis - dim_customer PK :=
VAR TotalRows =
    COUNTROWS(olist_customers_dataset)
VAR DistinctPK =
    DISTINCTCOUNT(olist_customers_dataset[customer_unique_id])
RETURN
TotalRows - DistinctPK
```

**Observed result:** `3345`

**Explanation**

The non-zero result is **expected** at this stage.

`olist_customers_dataset` is **order-scoped**, not customer-scoped.
The same `customer_unique_id` legitimately appears multiple times with different `customer_id` values, each tied to a different order.

At this point in the pipeline:

• the table is still a **source / bridge table**
• uniqueness of `customer_unique_id` is **not enforced yet**
• the measure is executed **before** customer dimension materialization

The violation disappears after the ANALYSIS-layer derivation of the customer dimension, where:

• 1 row = 1 `customer_unique_id`
• duplicates are collapsed deterministically

---

### dim_geography — PK check

**Logical PK:** `geolocation_zip_code_prefix`
**Measure name:** `Analysis - dim_geography PK`

```DAX
Analysis - dim_geography PK :=
VAR TotalRows =
    COUNTROWS(olist_geolocation_dataset)
VAR DistinctPK =
    DISTINCTCOUNT(olist_geolocation_dataset[geolocation_zip_code_prefix])
RETURN
TotalRows - DistinctPK
```

**Observed result:** `> 0`

**Explanation**

The non-zero result is **expected** on the raw geolocation dataset.

`olist_geolocation_dataset` contains **multiple records per ZIP code prefix**, associated with:

• different latitude / longitude values
• different city and state labels

At this stage:

• the dataset is still **pre-deduplication**
• ZIP code prefix is **not unique by construction**

The violation is resolved in the ANALYSIS layer by materializing a geography dimension at a declared grain, after which the PK check returns `0`.

---

### fact_orders — PK check

**Logical PK:** `order_id`
**Measure name:** `Analysis - fact_orders PK`

```DAX
Analysis - fact_orders PK :=
VAR TotalRows =
    COUNTROWS(olist_orders_dataset)
VAR DistinctPK =
    DISTINCTCOUNT(olist_orders_dataset[order_id])
RETURN
TotalRows - DistinctPK
```

**Observed result:** `0`

The order dataset respects its declared grain.
Each `order_id` is unique.

---

### fact_order_items — PK check

**Logical PK:** `(order_id, order_item_id)`
**Measure name:** `Analysis - fact_order_items PK`

```DAX
Analysis - fact_order_items PK :=
VAR TotalRows =
    COUNTROWS(olist_order_items_dataset)
VAR DistinctPK =
    COUNTROWS(
        SUMMARIZE(
            olist_order_items_dataset,
            olist_order_items_dataset[order_id],
            olist_order_items_dataset[order_item_id]
        )
    )
RETURN
TotalRows - DistinctPK
```

**Observed result:** `0`

The composite key uniquely identifies each order line item.

---

### fact_payments — PK check

**Logical PK:** `(order_id, payment_sequential)`
**Measure name:** `Analysis - fact_payments PK`

```DAX
Analysis - fact_payments PK :=
VAR TotalRows =
    COUNTROWS(olist_order_payments_dataset)
VAR DistinctPK =
    COUNTROWS(
        SUMMARIZE(
            olist_order_payments_dataset,
            olist_order_payments_dataset[order_id],
            olist_order_payments_dataset[payment_sequential]
        )
    )
RETURN
TotalRows - DistinctPK
```

**Observed result:** `0`

Each payment record is uniquely identified by order and sequence number.

---

### fact_reviews — PK check

**Logical PK:** `review_id`
**Measure name:** `Analysis - fact_reviews PK`

```DAX
Analysis - fact_reviews PK :=
VAR TotalRows =
    COUNTROWS(olist_order_reviews_dataset)
VAR DistinctPK =
    DISTINCTCOUNT(olist_order_reviews_dataset[review_id])
RETURN
TotalRows - DistinctPK
```

**Observed result:** `0`

Each review is uniquely identified.

---

### dim_seller — PK check

**Logical PK:** `seller_id`
**Measure name:** `Analysis - dim_seller PK`

```DAX
Analysis - dim_seller PK :=
VAR TotalRows =
    COUNTROWS(olist_sellers_dataset)
VAR DistinctPK =
    DISTINCTCOUNT(olist_sellers_dataset[seller_id])
RETURN
TotalRows - DistinctPK
```

**Observed result:** `0`

Seller identifiers are unique.

---

### dim_category — PK check

**Logical PK:** `product_category_name`
**Measure name:** `Analysis - dim_category PK`

```DAX
Analysis - dim_category PK :=
VAR TotalRows =
    COUNTROWS(product_category_name_translation)
VAR DistinctPK =
    DISTINCTCOUNT(product_category_name_translation[product_category_name])
RETURN
TotalRows - DistinctPK
```

**Observed result:** `0`

Each product category appears once.

---

## Summary of PK validation results

| Table / Concept  | PK status                              |
| ---------------- | -------------------------------------- |
| fact_orders      | Valid                                  |
| fact_order_items | Valid                                  |
| fact_payments    | Valid                                  |
| fact_reviews     | Valid                                  |
| dim_seller       | Valid                                  |
| dim_category     | Valid                                  |
| dim_customer     | Expected violation (pre-dimension)     |
| dim_geography    | Expected violation (pre-deduplication) |

---

This concludes the ANALYSIS-layer documentation for primary key validation and structural integrity checks.
