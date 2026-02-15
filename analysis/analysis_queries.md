# Olist E-commerce — Analysis Layer (Queries)

This document describes all queries implemented in the **ANALYSIS** layer using Power Query (M).

Only queries that introduce structural, temporal, or semantic logic at the analysis level are included.

All other queries reused directly from STAGING are intentionally omitted.

The model uses a single shared calendar dimension: `analysis_dim_date`.

---

## analysis_dim_date

**Role**
Single shared calendar dimension.

**Grain**
1 row = 1 calendar day.

```powerquery
let
    StartDate = #date(2016, 9, 1),
    EndDate   = #date(2018, 10, 31),

    DateList =
        List.Dates(
            StartDate,
            Duration.Days(EndDate - StartDate) + 1,
            #duration(1, 0, 0, 0)
        ),

    ToTable =
        Table.FromList(
            DateList,
            Splitter.SplitByNothing(),
            {"date"}
        ),

    TypedDate =
        Table.TransformColumnTypes(
            ToTable,
            {{"date", type date}}
        ),

    AddYear =
        Table.AddColumn(
            TypedDate,
            "year",
            each Date.Year([date]),
            Int64.Type
        ),

    AddMonth =
        Table.AddColumn(
            AddYear,
            "month",
            each Date.Month([date]),
            Int64.Type
        ),

    AddDay =
        Table.AddColumn(
            AddMonth,
            "day",
            each Date.Day([date]),
            Int64.Type
        ),

    AddYearMonth =
        Table.AddColumn(
            AddDay,
            "year_month",
            each Text.From([year]) & "-" & Text.PadStart(Text.From([month]), 2, "0"),
            type text
        ),

    AddYearMonthSort =
        Table.AddColumn(
            AddYearMonth,
            "year_month_sort",
            each [year] * 100 + [month],
            Int64.Type
        ),

    AddDateKey =
        Table.AddColumn(
            AddYearMonthSort,
            "date_key",
            each [year] * 10000 + [month] * 100 + [day],
            Int64.Type
        )
in
    AddDateKey
```

---

## analysis_fact_orders

**Grain**
1 row = 1 `order_id`

**Role**
Order-level fact table. Contains lifecycle dates and order status.

```powerquery
let
    Source =
        olist_orders_dataset,

    TypedColumns =
        Table.TransformColumnTypes(
            Source,
            {
                {"order_id", type text},
                {"customer_id", type text},
                {"order_status", type text},
                {"order_purchase_timestamp", type datetime},
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
            {"order_id", "customer_id", "order_status"}
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

    AddedPurchaseDate =
        Table.AddColumn(
            CleanText,
            "order_purchase_date",
            each if [order_purchase_timestamp] = null then null else Date.From([order_purchase_timestamp]),
            type date
        ),

    AddedDeliveredDate =
        Table.AddColumn(
            AddedPurchaseDate,
            "order_delivered_date",
            each if [order_delivered_customer_date] = null then null else Date.From([order_delivered_customer_date]),
            type date
        ),

    CastEstimatedToDate =
        Table.TransformColumns(
            AddedDeliveredDate,
            {
                {"order_estimated_delivery_date", each if _ = null then null else Date.From(_), type date}
            }
        ),

    RemovedTimestamps =
        Table.RemoveColumns(
            CastEstimatedToDate,
            {
                "order_purchase_timestamp",
                "order_delivered_carrier_date",
                "order_delivered_customer_date"
            }
        ),

    Deduplicated =
        Table.Distinct(
            RemovedTimestamps,
            {"order_id"}
        ),

    AddedPurchaseDateKey =
        Table.AddColumn(
            Deduplicated,
            "order_purchase_date_key",
            each
                if [order_purchase_date] = null
                then null
                else Date.Year([order_purchase_date]) * 10000
                   + Date.Month([order_purchase_date]) * 100
                   + Date.Day([order_purchase_date]),
            Int64.Type
        ),

    AddedDeliveredDateKey =
        Table.AddColumn(
            AddedPurchaseDateKey,
            "order_delivered_date_key",
            each
                if [order_delivered_date] = null
                then null
                else Date.Year([order_delivered_date]) * 10000
                   + Date.Month([order_delivered_date]) * 100
                   + Date.Day([order_delivered_date]),
            Int64.Type
        )
in
    AddedDeliveredDateKey
```

---

## analysis_fact_order_items

**Grain**
1 row = 1 (`order_id`, `order_item_id`)

**Role**
Line-level fact table for monetary and product-level analysis.

```powerquery
let
    Source =
        olist_order_items_dataset,

    TypedColumns =
        Table.TransformColumnTypes(
            Source,
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
                {"seller_id", each Text.Trim(Text.Clean(_))), type text}
            }
        ),

    NormalizedPrice =
        Table.TransformColumns(
            CleanText,
            {
                {"price", each Number.Round(_, 2), type number},
                {"freight_value", each Number.Round(_, 2), type number}
            }
        ),

    RemovedTimestamp =
        Table.RemoveColumns(
            NormalizedPrice,
            {"shipping_limit_date"}
        ),

    Deduplicated =
        Table.Distinct(
            RemovedTimestamp,
            {"order_id", "order_item_id"}
        ),

    MergedOrders =
        Table.NestedJoin(
            Deduplicated,
            {"order_id"},
            analysis_fact_orders,
            {"order_id"},
            "orders",
            JoinKind.LeftOuter
        ),

    ExpandedOrders =
        Table.ExpandTableColumn(
            MergedOrders,
            "orders",
            {
                "order_purchase_date_key",
                "order_delivered_date_key"
            },
            {
                "order_purchase_date_key",
                "order_delivered_date_key"
            }
        )
in
    ExpandedOrders
```

---

## analysis_fact_reviews

**Grain**
1 row = 1 `review_id`

**Role**
Review-level fact table. Supports satisfaction and SLA analysis.

```powerquery
let
    Source =
        olist_order_reviews_dataset,

    TypedColumns =
        Table.TransformColumnTypes(
            Source,
            {
                {"review_id", type text},
                {"order_id", type text},
                {"review_score", Int64.Type},
                {"review_creation_date", type datetime},
                {"review_answer_timestamp", type datetime}
            }
        ),

    EmptyToNull =
        Table.ReplaceValue(
            TypedColumns,
            "",
            null,
            Replacer.ReplaceValue,
            {
                "review_id",
                "order_id"
            }
        ),

    CleanText =
        Table.TransformColumns(
            EmptyToNull,
            {
                {"review_id", each Text.Trim(Text.Clean(_)), type text},
                {"order_id", each Text.Trim(Text.Clean(_)), type text}
            }
        ),

    CastCreationToDate =
        Table.TransformColumns(
            CleanText,
            {
                {
                    "review_creation_date",
                    each Date.From(_),
                    type date
                }
            }
        ),

    AddedCreationDateKey =
        Table.AddColumn(
            CastCreationToDate,
            "review_creation_date_key",
            each
                if [review_creation_date] = null
                then null
                else
                    Date.Year([review_creation_date]) * 10000
                    + Date.Month([review_creation_date]) * 100
                    + Date.Day([review_creation_date]),
            Int64.Type
        ),

    AddedReviewAnswerDate =
        Table.AddColumn(
            AddedCreationDateKey,
            "review_answer_date",
            each
                if [review_answer_timestamp] = null
                then null
                else Date.From([review_answer_timestamp]),
            type date
        ),

    AddedReviewAnswerDateKey =
        Table.AddColumn(
            AddedReviewAnswerDate,
            "review_answer_date_key",
            each
                if [review_answer_date] = null
                then null
                else
                    Date.Year([review_answer_date]) * 10000
                    + Date.Month([review_answer_date]) * 100
                    + Date.Day([review_answer_date]),
            Int64.Type
        ),

    RemovedColumns =
        Table.RemoveColumns(
            AddedReviewAnswerDateKey,
            {
                "review_answer_timestamp",
                "review_comment_title",
                "review_comment_message"
            }
        ),

    Deduplicated =
        Table.Distinct(
            RemovedColumns,
            {"review_id"}
        )
in
    Deduplicated
```

---

## analysis_dim_products

**Grain**
1 row = 1 `product_id`

**Role**
Product dimension enriched with derived physical metrics.

```powerquery
let
    Source =
        olist_products_dataset,

    MergedTranslation =
        Table.NestedJoin(
            Source,
            {"product_category_name"},
            analysis_dim_products_translation,
            {"product_category_name"},
            "translation",
            JoinKind.LeftOuter
        ),

    ExpandedTranslation =
        Table.ExpandTableColumn(
            MergedTranslation,
            "translation",
            {"product_category_name_english"},
            {"product_category_name_english"}
        ),

    RemovedOriginalCategory =
        Table.RemoveColumns(
            ExpandedTranslation,
            {"product_category_name"}
        ),

    RenamedEnglishCategory =
        Table.RenameColumns(
            RemovedOriginalCategory,
            {{"product_category_name_english", "product_category_name"}}
        ),

    RemovedUnusedColumns =
        Table.RemoveColumns(
            RenamedEnglishCategory,
            {
                "product_name_lenght",
                "product_description_lenght",
                "product_photos_qty"
            }
        ),

    AddedVolume =
        Table.AddColumn(
            RemovedUnusedColumns,
            "product_volume_cm3",
            each
                if [product_length_cm] = null
                    or [product_height_cm] = null
                    or [product_width_cm] = null
                then null
                else Number.Round(
                        [product_length_cm]
                        * [product_height_cm]
                        * [product_width_cm],
                        3
                     ),
            type number
        ),

    AddedVolumetricDensity =
        Table.AddColumn(
            AddedVolume,
            "product_volumetric_density",
            each
                if [product_weight_g] = null
                    or [product_volume_cm3] = null
                    or [product_volume_cm3] = 0
                then null
                else Number.Round(
                        [product_weight_g] / [product_volume_cm3],
                        3
                     ),
            type number
        ),

    ReorderedColumns =
        Table.ReorderColumns(
            AddedVolumetricDensity,
            {
                "product_id",
                "product_category_name",
                "product_weight_g",
                "product_length_cm",
                "product_height_cm",
                "product_width_cm",
                "product_volume_cm3",
                "product_volumetric_density"
            }
        )
in
    ReorderedColumns
```
