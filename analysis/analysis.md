# Olist E-commerce — Analysis Layer Documentation

This document describes the **ANALYSIS** layer of the Olist data model.

The ANALYSIS layer introduces:

* explicit semantic roles
* derived date keys (YYYYMMDD)
* controlled enrichment logic
* bridge-based customer resolution
* centralized temporal logic through a single shared calendar dimension

The current model reflects the simplified analytical asset defined in the semantic schema.

---

## Core Principles

* Fact table grains remain unchanged from STAGING.
* Surrogate keys are introduced only for temporal alignment.
* No artificial identifiers are created where natural keys are stable.
* All temporal logic is centralized through `analysis_dim_date`.
* Relationship integrity is validated through explicit DAX checks.

---

## analysis_fact_orders

**Source**
`olist_orders_dataset` (STAGING)

**Grain**
1 row = 1 `order_id`

### Structural Notes

* Original timestamps are used only for date derivation.
* Timestamp columns are removed after transformation.
* Grain is enforced via `order_id` deduplication.

### Derived Date Columns

* `order_purchase_date`
* `order_delivered_date`

`order_estimated_delivery_date` is cast to `date`.

### Date Keys (YYYYMMDD)

* `order_purchase_date_key`
* `order_delivered_date_key`

These keys connect to `analysis_dim_date`.

No event-quality flags are present in the current model.

---

## analysis_fact_order_items

**Source**
`olist_order_items_dataset` (STAGING)

**Grain**
1 row = 1 (`order_id`, `order_item_id`)

### Structural Logic

* Monetary columns `price` and `freight_value` are rounded to 2 decimals.
* `shipping_limit_date` is removed.
* Grain enforced via composite deduplication.

### Inherited Date Keys

The table performs a left join to `analysis_fact_orders` to inherit:

* `order_purchase_date_key`
* `order_delivered_date_key`

This preserves item-level alignment with order-level temporal context.

No shipping flags or time-splitting columns are present in the current model.

---

## analysis_fact_reviews

**Source**
`olist_order_reviews_dataset` (STAGING)

**Grain**
1 row = 1 `review_id`

### Derived Columns

* `review_creation_date` (cast to date)
* `review_creation_date_key` (YYYYMMDD)
* `review_answer_date`
* `review_answer_date_key`

Raw timestamp columns are removed.

Unused textual fields are removed:

* `review_comment_title`
* `review_comment_message`

Grain is enforced via `review_id` deduplication.

No explicit review-quality flags are present.

---

## analysis_dim_date

**Role**
Single shared calendar dimension.

**Grain**
1 row = 1 calendar day.

**Range**
2016-09-01 → 2018-10-31

**Primary Key**
`date_key` (YYYYMMDD)

**Attributes**

* `date`
* `year`
* `month`
* `day`
* `year_month`
* `year_month_sort`

---

## analysis_dim_customer

**Grain**
1 row = 1 `customer_unique_id`

Represents real individuals across potentially multiple customer accounts.

---

## analysis_bridge_customer_account

**Grain**
1 row = 1 (`customer_id`, `customer_unique_id`)

Resolves transactional `customer_id` from orders to analytical `customer_unique_id`.

---

## analysis_dim_products

**Grain**
1 row = 1 `product_id`

### Enrichment Logic

Derived physical metrics:

* `product_volume_cm3`
* `product_volumetric_density`

Category translation is resolved at analysis level.

---

## Date Relationships

Active relationship:

* `analysis_fact_orders[order_purchase_date_key]` → `analysis_dim_date[date_key]`

Inactive relationships:

* `analysis_fact_orders[order_delivered_date_key]`
* `analysis_fact_order_items[order_purchase_date_key]`
* `analysis_fact_order_items[order_delivered_date_key]`
* `analysis_fact_reviews[review_creation_date_key]`
* `analysis_fact_reviews[review_answer_date_key]`

Temporal switching is handled in DAX using `USERELATIONSHIP`.

---

## Removed Elements (Not Present in Current Model)

The following structures are not part of the current analysis schema:

* payments fact
* payment type dimension
* seller dimension
* geolocation dimension
* geography surrogate keys
* event-quality flags
* shipping time-splitting columns

The documentation reflects only the active semantic model.

---

## Validation Status

* Fact grains enforced.
* Composite keys respected.
* Date keys generated consistently.
* All primary key and referential integrity checks return 0.
* No orphan records detected.
* Model aligned with a single shared calendar architecture.
