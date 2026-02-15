# Olist E-commerce — Staging Layer Documentation

This document describes the **STAGING layer** of the Olist e-commerce project.

The purpose of the STAGING layer is to **ingest raw CSV source files** and apply only minimal, non-semantic transformations required to make the data structurally usable in downstream layers.

The following operations are applied consistently across datasets:

• column header promotion
• explicit column typing
• minimal text cleaning (trim, clean, case normalization where present)
• empty string → `null` normalization
• structural deduplication at the declared grain

No semantic transformation, enrichment, key derivation, or business logic is applied in this layer.

---

## olist_customers_dataset (STAGING)

**Source**
`olist_customers_dataset.csv`

**Grain**
1 row = 1 `(customer_id, customer_unique_id)`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_customers_dataset.csv"),
            [Delimiter=",", Columns=5, Encoding=1252, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"customer_id", type text},
                {"customer_unique_id", type text},
                {"customer_zip_code_prefix", type text},
                {"customer_city", type text},
                {"customer_state", type text}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "customer_id",
                "customer_unique_id",
                "customer_zip_code_prefix",
                "customer_city",
                "customer_state"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"customer_id", each Text.Trim(Text.Clean(_)), type text},
                {"customer_unique_id", each Text.Trim(Text.Clean(_)), type text},
                {"customer_zip_code_prefix", each Text.Trim(Text.Clean(_)), type text},
                {"customer_city", each Text.Lower(Text.Trim(Text.Clean(_))), type text},
                {"customer_state", each Text.Upper(Text.Trim(Text.Clean(_))), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText, {"customer_id", "customer_unique_id"})
in
    Deduplicated
```

---

## olist_geolocation_dataset (STAGING)

**Source**
`olist_geolocation_dataset.csv`

**Grain**
1 row = 1 raw geolocation observation

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_geolocation_dataset.csv"),
            [Delimiter=",", Columns=5, Encoding=65001, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"geolocation_zip_code_prefix", type text},
                {"geolocation_lat", type text},
                {"geolocation_lng", type text},
                {"geolocation_city", type text},
                {"geolocation_state", type text}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "geolocation_zip_code_prefix",
                "geolocation_city",
                "geolocation_state"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"geolocation_zip_code_prefix", each Text.Trim(Text.Clean(_)), type text},
                {"geolocation_city", each Text.Lower(Text.Trim(Text.Clean(_))), type text},
                {"geolocation_state", each Text.Upper(Text.Trim(Text.Clean(_))), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText)
in
    Deduplicated
```

---

## olist_order_items_dataset (STAGING)

**Source**
`olist_order_items_dataset.csv`

**Grain**
1 row = 1 `(order_id, order_item_id)`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_order_items_dataset.csv"),
            [Delimiter=",", Columns=7, Encoding=1252, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"order_id", type text},
                {"order_item_id", Int64.Type},
                {"product_id", type text},
                {"seller_id", type text},
                {"shipping_limit_date", type datetime},
                {"price", type number},
                {"freight_value", type number}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "order_id",
                "product_id",
                "seller_id"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"order_id", each Text.Trim(Text.Clean(_)), type text},
                {"product_id", each Text.Trim(Text.Clean(_)), type text},
                {"seller_id", each Text.Trim(Text.Clean(_)), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText, {"order_id", "order_item_id"})
in
    Deduplicated
```

---

## olist_order_payments_dataset (STAGING)

**Source**
`olist_order_payments_dataset.csv`

**Grain**
1 row = 1 `(order_id, payment_sequential)`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_order_payments_dataset.csv"),
            [Delimiter=",", Columns=5, Encoding=1252, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"order_id", type text},
                {"payment_sequential", Int64.Type},
                {"payment_type", type text},
                {"payment_installments", Int64.Type},
                {"payment_value", type number}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "order_id",
                "payment_type"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"order_id", each Text.Trim(Text.Clean(_)), type text},
                {"payment_type", each Text.Lower(Text.Trim(Text.Clean(_))), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText, {"order_id", "payment_sequential"})
in
    Deduplicated
```

---

## olist_order_reviews_dataset (STAGING)

**Source**
`olist_order_reviews_dataset.csv`

**Grain**
1 row = 1 `review_id`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_order_reviews_dataset.csv"),
            [
                Delimiter = ",",
                Columns = 7,
                Encoding = 65001,
                QuoteStyle = QuoteStyle.Csv
            ]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    EmptyToNull =
        Table.ReplaceValue(
            PromotedHeaders,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "review_id",
                "order_id",
                "review_comment_title",
                "review_comment_message",
                "review_creation_date",
                "review_answer_timestamp"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"review_id", each if _ = null then null else Text.Trim(Text.Clean(_)), type text},
                {"order_id", each if _ = null then null else Text.Trim(Text.Clean(_)), type text},
                {"review_comment_title", each if _ = null then null else Text.Trim(Text.Clean(_)), type text},
                {"review_comment_message", each if _ = null then null else Text.Trim(Text.Clean(_)), type text}
            }
        ),

    TypedColumns =
        Table.TransformColumnTypes(
            CleanText,
            {
                {"review_id", type text},
                {"order_id", type text},
                {"review_score", Int64.Type},
                {"review_comment_title", type text},
                {"review_comment_message", type text}
            }
        ),

    ParseDates =
        Table.TransformColumns(
            TypedColumns,
            {
                {"review_creation_date", each try DateTime.FromText(_, "en-US") otherwise null, type datetime},
                {"review_answer_timestamp", each try DateTime.FromText(_, "en-US") otherwise null, type datetime}
            }
        ),

    Deduplicated =
        Table.Distinct(ParseDates, {"review_id"})
in
    Deduplicated
```

---

## olist_orders_dataset (STAGING)

**Source**
`olist_orders_dataset.csv`

**Grain**
1 row = 1 `order_id`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_orders_dataset.csv"),
            [Delimiter=",", Columns=8, Encoding=1252, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"order_id", type text},
                {"customer_id", type text},
                {"order_status", type text},
                {"order_purchase_timestamp", type datetime},
                {"order_approved_at", type datetime},
                {"order_delivered_carrier_date", type datetime},
                {"order_delivered_customer_date", type datetime},
                {"order_estimated_delivery_date", type datetime}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "order_id",
                "customer_id",
                "order_status",
                "order_purchase_timestamp",
                "order_approved_at",
                "order_delivered_carrier_date",
                "order_delivered_customer_date",
                "order_estimated_delivery_date"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"order_id", each Text.Trim(Text.Clean(_)), type text},
                {"customer_id", each Text.Trim(Text.Clean(_)), type text},
                {"order_status", each Text.Lower(Text.Trim(Text.Clean(_))), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText, {"order_id"})
in
    Deduplicated
```

---

## olist_products_dataset (STAGING)

**Source**
`olist_products_dataset.csv`

**Grain**
1 row = 1 `product_id`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_products_dataset.csv"),
            [Delimiter=",", Columns=9, Encoding=1252, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"product_id", type text},
                {"product_category_name", type text},
                {"product_name_lenght", Int64.Type},
                {"product_description_lenght", Int64.Type},
                {"product_photos_qty", Int64.Type},
                {"product_weight_g", Int64.Type},
                {"product_length_cm", Int64.Type},
                {"product_height_cm", Int64.Type},
                {"product_width_cm", Int64.Type}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "product_id",
                "product_category_name"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"product_id", each Text.Trim(Text.Clean(_)), type text},
                {"product_category_name", each Text.Lower(Text.Trim(Text.Clean(_))), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText, {"product_id"})
in
    Deduplicated
```

---

## olist_sellers_dataset (STAGING)

**Source**
`olist_sellers_dataset.csv`

**Grain**
1 row = 1 `seller_id`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\olist_sellers_dataset.csv"),
            [Delimiter=",", Columns=4, Encoding=1252, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"seller_id", type text},
                {"seller_zip_code_prefix", type text},
                {"seller_city", type text},
                {"seller_state", type text}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "seller_id",
                "seller_zip_code_prefix",
                "seller_city",
                "seller_state"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"seller_id", each Text.Trim(Text.Clean(_)), type text},
                {"seller_zip_code_prefix", each Text.Trim(Text.Clean(_)), type text},
                {"seller_city", each Text.Lower(Text.Trim(Text.Clean(_))), type text},
                {"seller_state", each Text.Upper(Text.Trim(Text.Clean(_))), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText, {"seller_id"})
in
    Deduplicated
```

---

## product_category_name_translation (STAGING)

**Source**
`product_category_name_translation.csv`

**Grain**
1 row = 1 `product_category_name`

```powerquery
let
    Source =
        Csv.Document(
            File.Contents("C:\Users\Lenovo\Documents\GitHub\olist_ecommerce_powerbi\staging\product_category_name_translation.csv"),
            [Delimiter=",", Columns=2, Encoding=1252, QuoteStyle=QuoteStyle.None]
        ),

    PromotedHeaders =
        Table.PromoteHeaders(Source, [PromoteAllScalars=true]),

    TypedColumns =
        Table.TransformColumnTypes(
            PromotedHeaders,
            {
                {"product_category_name", type text},
                {"product_category_name_english", type text}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "product_category_name",
                "product_category_name_english"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"product_category_name", each Text.Lower(Text.Trim(Text.Clean(_))), type text},
                {"product_category_name_english", each Text.Lower(Text.Trim(Text.Clean(_))), type text}
            }
        ),

    Deduplicated =
        Table.Distinct(CleanText, {"product_category_name"})
in
    Deduplicated
```