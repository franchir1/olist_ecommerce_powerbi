## Data Dictionary

This section provides a technical description of each source table and field as present in the raw Olist datasets.
The purpose is to define **field-level meaning and scope**, without interpretation, transformation logic, or analytical assumptions.

---

### olist_customers_dataset

**Table meaning**
Customer records as captured at order time.
The table does not represent a stable customer master.

**Grain**
1 row = 1 customer record associated with a single order.

| Field                    | Meaning                                                           |
| ------------------------ | ----------------------------------------------------------------- |
| customer_id              | Surrogate identifier of the customer record, scoped to an order   |
| customer_unique_id       | Persistent identifier of the same customer across multiple orders |
| customer_zip_code_prefix | Customer ZIP code prefix (first digits only)                      |
| customer_city            | Customer city as declared at order time                           |
| customer_state           | Customer state as declared at order time                          |

---

### olist_geolocation_dataset

**Table meaning**
Reference table providing geographic information by ZIP code prefix.

**Grain**
1 row = 1 ZIP code prefix and coordinate pair (not unique).

| Field                       | Meaning                                       |
| --------------------------- | --------------------------------------------- |
| geolocation_zip_code_prefix | ZIP code prefix used for geolocation          |
| geolocation_lat             | Latitude associated with the ZIP code prefix  |
| geolocation_lng             | Longitude associated with the ZIP code prefix |
| geolocation_city            | City associated with the ZIP code prefix      |
| geolocation_state           | State associated with the ZIP code prefix     |

---

### olist_orders_dataset

**Table meaning**
Order-level transactional data.

**Grain**
1 row = 1 order.

| Field                         | Meaning                                             |
| ----------------------------- | --------------------------------------------------- |
| order_id                      | Unique order identifier                             |
| customer_id                   | Customer identifier (FK to olist_customers_dataset) |
| order_status                  | Current order status                                |
| order_purchase_timestamp      | Timestamp when the order was placed                 |
| order_approved_at             | Timestamp when the order was approved               |
| order_delivered_carrier_date  | Date when the order was handed to the carrier       |
| order_delivered_customer_date | Date when the order was delivered to the customer   |
| order_estimated_delivery_date | Estimated delivery date                             |

---

### olist_order_items_dataset

**Table meaning**
Order line items representing products sold within orders.

**Grain**
1 row = 1 product item within an order.

| Field               | Meaning                                 |
| ------------------- | --------------------------------------- |
| order_id            | Order identifier (FK)                   |
| order_item_id       | Sequential item number within the order |
| product_id          | Product identifier                      |
| seller_id           | Seller identifier                       |
| shipping_limit_date | Seller shipping deadline                |
| price               | Item price excluding freight            |
| freight_value       | Freight cost charged for the item       |

---

### olist_order_payments_dataset

**Table meaning**
Payment records associated with orders.

**Grain**
1 row = 1 payment transaction related to an order.

| Field                | Meaning                                           |
| -------------------- | ------------------------------------------------- |
| order_id             | Order identifier                                  |
| payment_sequential   | Sequential number of the payment within the order |
| payment_type         | Payment method                                    |
| payment_installments | Number of installments used                       |
| payment_value        | Amount paid in this payment record                |

---

### olist_order_reviews_dataset

**Table meaning**
Customer feedback provided after order completion.

**Grain**
1 row = 1 review per order.

| Field                   | Meaning                                     |
| ----------------------- | ------------------------------------------- |
| review_id               | Unique review identifier                    |
| order_id                | Order identifier associated with the review |
| review_score            | Review score (1–5)                          |
| review_comment_title    | Review title written by the customer        |
| review_comment_message  | Review text written by the customer         |
| review_creation_date    | Date when the review was created            |
| review_answer_timestamp | Timestamp of the seller reply               |

---

### olist_products_dataset

**Table meaning**
Product reference information.

**Grain**
1 row = 1 product.

| Field                      | Meaning                                 |
| -------------------------- | --------------------------------------- |
| product_id                 | Unique product identifier               |
| product_category_name      | Product category name (Portuguese)      |
| product_name_lenght        | Product name length (characters)        |
| product_description_lenght | Product description length (characters) |
| product_photos_qty         | Number of product photos                |
| product_weight_g           | Product weight in grams                 |
| product_length_cm          | Product length in centimeters           |
| product_height_cm          | Product height in centimeters           |
| product_width_cm           | Product width in centimeters            |

---

### olist_sellers_dataset

**Table meaning**
Seller reference information.

**Grain**
1 row = 1 seller.

| Field                  | Meaning                  |
| ---------------------- | ------------------------ |
| seller_id              | Unique seller identifier |
| seller_zip_code_prefix | Seller ZIP code prefix   |
| seller_city            | Seller city              |
| seller_state           | Seller state             |

---

### product_category_name_translation

**Table meaning**
Mapping table translating product categories from Portuguese to English.

**Grain**
1 row = 1 product category.

| Field                         | Meaning                                     |
| ----------------------------- | ------------------------------------------- |
| product_category_name         | Original product category name (Portuguese) |
| product_category_name_english | Product category name in English            |
