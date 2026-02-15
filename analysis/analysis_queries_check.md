## Analysis Layer — Data Quality and Key Integrity Checks

This section documents the DAX measures implemented to validate primary key integrity and mandatory relationships within the **ANALYSIS** layer.

All checks are executed directly on the analysis tables and reflect the current semantic model:

* `analysis_fact_orders`
* `analysis_fact_order_items`
* `analysis_fact_reviews`
* `analysis_dim_date`
* `analysis_dim_customer`
* `analysis_bridge_customer_account`
* `analysis_dim_products`

All measures listed below return a value equal to **0**.

---

### Analysis – dim_date PK

Validates primary key uniqueness for `analysis_dim_date` using `date_key`.

```DAX
Analysis - dim_date PK = 
VAR TotalRows =
    COUNTROWS(analysis_dim_date)
VAR DistinctPK =
    DISTINCTCOUNT(analysis_dim_date[date_key])
RETURN
TotalRows - DistinctPK
```

---

### Analysis – dim_customer PK

Validates primary key uniqueness for `analysis_dim_customer` using `customer_unique_id`.

```DAX
Analysis - dim_customer PK = 
VAR TotalRows =
    COUNTROWS(analysis_dim_customer)
VAR DistinctPK =
    DISTINCTCOUNT(analysis_dim_customer[customer_unique_id])
RETURN
TotalRows - DistinctPK
```

---

### Analysis – bridge_customer_account PK

Validates composite primary key uniqueness for `analysis_bridge_customer_account` using (`customer_id`, `customer_unique_id`).

```DAX
Analysis - bridge_customer_account PK = 
VAR TotalRows =
    COUNTROWS(analysis_bridge_customer_account)
VAR DistinctPK =
    COUNTROWS(
        SUMMARIZE(
            analysis_bridge_customer_account,
            analysis_bridge_customer_account[customer_id],
            analysis_bridge_customer_account[customer_unique_id]
        )
    )
RETURN
TotalRows - DistinctPK
```

---

### Analysis – dim_products PK

Validates primary key uniqueness for `analysis_dim_products` using `product_id`.

```DAX
Analysis - dim_products PK = 
VAR TotalRows =
    COUNTROWS(analysis_dim_products)
VAR DistinctPK =
    DISTINCTCOUNT(analysis_dim_products[product_id])
RETURN
TotalRows - DistinctPK
```

---

### Analysis – fact_orders PK

Validates primary key uniqueness for `analysis_fact_orders` using `order_id`.

```DAX
Analysis - fact_orders PK = 
VAR TotalRows =
    COUNTROWS(analysis_fact_orders)
VAR DistinctPK =
    DISTINCTCOUNT(analysis_fact_orders[order_id])
RETURN
TotalRows - DistinctPK
```

---

### Analysis – fact_order_items PK

Validates composite primary key uniqueness for `analysis_fact_order_items` using (`order_id`, `order_item_id`).

```DAX
Analysis - fact_order_items PK = 
VAR TotalRows =
    COUNTROWS(analysis_fact_order_items)
VAR DistinctPK =
    COUNTROWS(
        SUMMARIZE(
            analysis_fact_order_items,
            analysis_fact_order_items[order_id],
            analysis_fact_order_items[order_item_id]
        )
    )
RETURN
TotalRows - DistinctPK
```

---

### Analysis – fact_reviews PK

Validates primary key uniqueness for `analysis_fact_reviews` using `review_id`.

```DAX
Analysis - fact_reviews PK = 
VAR TotalRows =
    COUNTROWS(analysis_fact_reviews)
VAR DistinctPK =
    DISTINCTCOUNT(analysis_fact_reviews[review_id])
RETURN
TotalRows - DistinctPK
```

---

### Analysis – orders missing customer

Counts orders in `analysis_fact_orders` where the `customer_id` does not resolve to a valid `customer_unique_id` through the bridge and customer dimension.

```DAX
Analysis - orders missing customer = 
COUNTROWS(
    FILTER(
        analysis_fact_orders,
        ISBLANK(
            RELATED(analysis_bridge_customer_account[customer_unique_id])
        )
            ||
        ISBLANK(
            RELATED(analysis_dim_customer[customer_unique_id])
        )
    )
)
```

This check validates mandatory referential integrity between:

* `analysis_fact_orders`
* `analysis_bridge_customer_account`
* `analysis_dim_customer`

---

### Analysis – order_items missing order

Counts rows in `analysis_fact_order_items` where no matching `order_id` exists in `analysis_fact_orders`.

```DAX
Analysis - order_items missing order = 
COUNTROWS(
    FILTER(
        analysis_fact_order_items,
        ISBLANK(
            RELATED(analysis_fact_orders[order_id])
        )
    )
)
```

---

### Analysis – order_items missing product

Counts rows in `analysis_fact_order_items` where `product_id` does not resolve to `analysis_dim_products`.

```DAX
Analysis - order_items missing product = 
COUNTROWS(
    FILTER(
        analysis_fact_order_items,
        ISBLANK(
            RELATED(analysis_dim_products[product_id])
        )
    )
)
```

---

## Validation Result Summary

All listed measures return **0**.

This confirms that:

* Primary keys are unique across all dimensions and fact tables.
* Composite keys are structurally respected.
* Mandatory relationships resolve correctly.
* No orphan records are present in the analytical model.
