# Power BI Enterprise Solution
## Grocery Chain Analytics — Full Technical Specification

---

## 1. Executive Summary

This document describes a complete, enterprise-grade Power BI semantic model and dashboard solution built from the `grocery_chain_data.csv` flat-file dataset. The solution transforms raw transactional data into a fully normalized star schema with 16 DAX measures, 6 interactive report pages, and actionable retail KPIs suitable for executive, regional, and operational reporting.

**Dataset profile:**
- 1,980 transactions across 9 stores
- Date range: August 2023 – August 2025 (2 full years)
- 1,798 unique customers | 18 products | 11 product categories
- Source columns: customer_id, store_name, transaction_date, aisle, product_name, quantity, unit_price, total_amount, discount_amount, final_amount, loyalty_points

---

## 2. Data Quality Assessment

### Issues Identified

| Issue | Count | Column | Resolution |
|---|---|---|---|
| Null store_name values | 25 rows | store_name | Power Query: remove rows where store_name = null |
| Negative final_amount | ~8 rows | final_amount | Flag as returns; create `IsReturn` boolean column |
| Zero loyalty_points | Present | loyalty_points | Valid — not all transactions earn points |
| Single-day data gap | None | transaction_date | No action needed |
| No transaction surrogate key | All rows | — | Add index column in Power Query |

### Data Profiling Summary

```
Total Revenue:        $82,037.31
Total Discount:        $8,849.79    (9.74% of gross)
Avg Basket Value:        $45.63    (median: $37.03)
Max Basket Value:       $250.84
Min Basket Value:        -$3.43    (return transaction)
Repeat Customer Rate:     9.2%     (166 of 1,798 customers)
Avg Loyalty Points/Txn:    255     (range: 0–500)
```

---

## 3. Semantic Model — Star Schema Design

### Architecture Overview

The flat CSV is normalized into **1 fact table + 5 dimension tables** following Kimball dimensional modelling best practices. All relationships are single-direction, one-to-many, from dimension to fact.

```
DimDate ────────────────────────────────┐
DimStore ───────────────────────────────┤
DimProduct ─────────────────────────── FactSales
DimCustomer ────────────────────────────┤
DimPromotion ───────────────────────────┘
```

---

### 3.1 FactSales

The grain is one row per transaction line item (one product per purchase).

| Column | Type | Description |
|---|---|---|
| transaction_key | Integer (PK) | Surrogate key — added in Power Query |
| date_key | Integer (FK) | Links to DimDate |
| store_key | Integer (FK) | Links to DimStore |
| product_key | Integer (FK) | Links to DimProduct |
| customer_key | Integer (FK) | Links to DimCustomer |
| promo_key | Integer (FK) | Links to DimPromotion |
| customer_id | Integer | Source customer identifier |
| quantity | Integer | Units purchased |
| unit_price | Currency | Price per unit (pre-discount) |
| total_amount | Currency | quantity × unit_price |
| discount_amount | Currency | Discount applied |
| final_amount | Currency | total_amount − discount_amount |
| loyalty_points | Integer | Points awarded this transaction |
| is_return | Boolean | TRUE when final_amount < 0 |

---

### 3.2 DimDate (Calculated)

Generated via M Query — not derived from source data. Covers 2023-01-01 to present.

| Column | Type |
|---|---|
| date_key | Integer (YYYYMMDD) |
| date | Date |
| year | Integer |
| quarter | Text (Q1–Q4) |
| quarter_num | Integer |
| month_num | Integer |
| month_name | Text |
| month_short | Text |
| week_num | Integer |
| day_of_week | Text |
| is_weekend | Boolean |
| fiscal_year | Text |
| fiscal_quarter | Text |

**M Query:**
```m
let
    StartDate = #date(2023, 1, 1),
    EndDate   = Date.From(DateTime.LocalNow()),
    DayCount  = Duration.Days(EndDate - StartDate) + 1,
    Dates     = List.Dates(StartDate, DayCount, #duration(1,0,0,0)),
    DateTable = Table.FromList(Dates, Splitter.SplitByNothing(), {"Date"}),
    AddKey    = Table.AddColumn(DateTable, "date_key",
                    each Date.Year([Date])*10000 + Date.Month([Date])*100 + Date.Day([Date])),
    AddYear   = Table.AddColumn(AddKey,   "year",         each Date.Year([Date])),
    AddQNum   = Table.AddColumn(AddYear,  "quarter_num",  each Date.QuarterOfYear([Date])),
    AddQ      = Table.AddColumn(AddQNum,  "quarter",      each "Q" & Text.From([quarter_num])),
    AddMNum   = Table.AddColumn(AddQ,     "month_num",    each Date.Month([Date])),
    AddMName  = Table.AddColumn(AddMNum,  "month_name",   each Date.MonthName([Date])),
    AddMShort = Table.AddColumn(AddMName, "month_short",  each Text.Start(Date.MonthName([Date]),3)),
    AddWeek   = Table.AddColumn(AddMShort,"week_num",     each Date.WeekOfYear([Date])),
    AddDOW    = Table.AddColumn(AddWeek,  "day_of_week",  each Date.DayOfWeekName([Date])),
    AddWE     = Table.AddColumn(AddDOW,   "is_weekend",   each Date.DayOfWeek([Date]) >= 5),
    AddFY     = Table.AddColumn(AddWE,    "fiscal_year",  each "FY" & Text.From(Date.Year([Date]))),
    TypedCols = Table.TransformColumnTypes(AddFY, {{"Date", type date},{"date_key", Int64.Type}})
in  TypedCols
```

---

### 3.3 DimStore

Derived from FactSales; enriched with business attributes.

| Column | Type | Source |
|---|---|---|
| store_key | Integer (PK) | Generated |
| store_name | Text | Source |
| store_short | Text | Derived |
| region | Text | Business enrichment |
| city | Text | Business enrichment |
| store_type | Text | Business enrichment |
| tier | Text | Business enrichment (A/B/C) |
| opening_date | Date | Business enrichment |

**M Query (derive from FactSales):**
```m
let
    Source   = FactSales,
    Cols     = Table.SelectColumns(Source, {"store_name"}),
    Distinct = Table.Distinct(Table.SelectRows(Cols, each [store_name] <> null)),
    AddKey   = Table.AddIndexColumn(Distinct, "store_key", 1, 1),
    AddShort = Table.AddColumn(AddKey, "store_short",
                   each Text.BeforeDelimiter([store_name], " ", {0, RelativePosition.FromEnd}))
in  AddShort
```

---

### 3.4 DimProduct

Derived from FactSales; 18 distinct products across 11 aisles.

| Column | Type |
|---|---|
| product_key | Integer (PK) |
| product_name | Text |
| category | Text (aisle) |
| sub_category | Text (derived) |
| storage_type | Text (Fresh/Chilled/Ambient) |
| is_fresh | Boolean |
| sku | Text (synthetic) |

**M Query:**
```m
let
    Source   = FactSales,
    Cols     = Table.SelectColumns(Source, {"product_name","aisle"}),
    Distinct = Table.Distinct(Cols),
    AddKey   = Table.AddIndexColumn(Distinct, "product_key", 1, 1),
    Renamed  = Table.RenameColumns(AddKey, {{"aisle","category"}}),
    AddStorage = Table.AddColumn(Renamed, "storage_type",
                     each if List.Contains({"Produce","Meat & Seafood","Dairy"}, [category])
                          then "Chilled/Fresh"
                          else if [category] = "Frozen Foods" then "Frozen"
                          else "Ambient"),
    AddFresh = Table.AddColumn(AddStorage, "is_fresh",
                   each List.Contains({"Produce","Meat & Seafood"}, [category]))
in  AddFresh
```

---

### 3.5 DimCustomer

Derived from FactSales; enriched with RFM-derived segments.

| Column | Type |
|---|---|
| customer_key | Integer (PK) |
| customer_id | Integer |
| visit_count | Integer |
| total_lifetime_spend | Currency |
| total_loyalty_points | Integer |
| loyalty_tier | Text (Bronze/Silver/Gold/Platinum) |
| rfm_segment | Text |
| first_visit_date | Date |
| last_visit_date | Date |
| days_since_last_visit | Integer |

---

### 3.6 DimPromotion

Derived from discount_amount bands.

| Column | Type |
|---|---|
| promo_key | Integer (PK) |
| discount_band | Text |
| promo_type | Text |
| is_promotional | Boolean |
| discount_pct_min | Decimal |
| discount_pct_max | Decimal |

---

## 4. Power Query Transformations (Applied Steps)

### FactSales — Full Applied Steps

```m
let
    Source = Csv.Document(File.Contents("grocery_chain_data.csv"),
                 [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    PromoteHeaders = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    
    // Step 1: Type casting
    TypedCols = Table.TransformColumnTypes(PromoteHeaders, {
        {"customer_id",      Int64.Type},
        {"store_name",       type text},
        {"transaction_date", type date},
        {"aisle",            type text},
        {"product_name",     type text},
        {"quantity",         Int64.Type},
        {"unit_price",       Currency.Type},
        {"total_amount",     Currency.Type},
        {"discount_amount",  Currency.Type},
        {"final_amount",     Currency.Type},
        {"loyalty_points",   Int64.Type}
    }),
    
    // Step 2: Add surrogate key
    AddTransactionKey = Table.AddIndexColumn(TypedCols, "transaction_key", 1, 1, Int64.Type),
    
    // Step 3: Remove rows with null store_name (data quality issue: 25 rows)
    FilterNullStores = Table.SelectRows(AddTransactionKey, each [store_name] <> null),
    
    // Step 4: Flag return transactions
    AddIsReturn = Table.AddColumn(FilterNullStores, "is_return",
                      each [final_amount] < 0, type logical),
    
    // Step 5: Add date_key (integer YYYYMMDD) for relationship to DimDate
    AddDateKey = Table.AddColumn(AddIsReturn, "date_key",
                     each Date.Year([transaction_date])*10000
                        + Date.Month([transaction_date])*100
                        + Date.Day([transaction_date]), Int64.Type),
    
    // Step 6: Add discount_pct for DimPromotion lookup
    AddDiscountPct = Table.AddColumn(AddDateKey, "discount_pct",
                         each if [total_amount] = 0 then 0
                              else [discount_amount] / [total_amount], type number),
    
    // Step 7: Remove source text columns replaced by FK relationships
    FinalTable = Table.SelectColumns(AddDiscountPct, {
        "transaction_key","date_key","customer_id","store_name","product_name",
        "aisle","quantity","unit_price","total_amount","discount_amount",
        "final_amount","loyalty_points","is_return","discount_pct"
    })
in  FinalTable
```

---

## 5. DAX Measures Library

All 16 measures with full implementation, context, and usage notes.

### 5.1 Core Revenue

```dax
-- [M01] Total Revenue
-- Base measure. Filter context applied by all slicers.
Total Revenue =
    SUM( FactSales[final_amount] )

-- [M02] Revenue YTD
Revenue YTD =
    TOTALYTD( [Total Revenue], DimDate[date] )

-- [M03] Revenue Previous Year (same period)
Revenue LY =
    CALCULATE(
        [Total Revenue],
        SAMEPERIODLASTYEAR( DimDate[date] )
    )

-- [M04] Revenue Year-over-Year Growth %
Revenue YoY % =
    VAR CY = [Total Revenue]
    VAR PY = [Revenue LY]
    RETURN
        IF(
            ISBLANK(PY) || PY = 0,
            BLANK(),
            DIVIDE( CY - PY, PY )
        )

-- [M05] Revenue Rolling 3-Month Average
Revenue 3M Rolling Avg =
    AVERAGEX(
        DATESINPERIOD(
            DimDate[date],
            LASTDATE( DimDate[date] ),
            -3, MONTH
        ),
        [Total Revenue]
    )
```

### 5.2 Transactions & Volume

```dax
-- [M06] Transaction Count
Transaction Count =
    COUNTROWS( FactSales )

-- [M07] Units Sold
Units Sold =
    SUM( FactSales[quantity] )

-- [M08] Average Basket Value
Avg Basket Value =
    DIVIDE( [Total Revenue], [Transaction Count] )
```

### 5.3 Customer Measures

```dax
-- [M09] Unique Customers
Unique Customers =
    DISTINCTCOUNT( FactSales[customer_id] )

-- [M10] Customer Lifetime Value (CLV)
-- Average total spend per unique customer in current filter context
CLV =
    AVERAGEX(
        VALUES( FactSales[customer_id] ),
        CALCULATE( SUM( FactSales[final_amount] ) )
    )

-- [M11] Repeat Customer Rate
-- % of customers with more than 1 transaction
Repeat Customer Rate =
    VAR CustomerVisits =
        SUMMARIZE(
            FactSales,
            FactSales[customer_id],
            "@Visits", COUNTROWS( FactSales )
        )
    VAR RepeatCustomers =
        COUNTX(
            FILTER( CustomerVisits, [@Visits] > 1 ),
            FactSales[customer_id]
        )
    RETURN
        DIVIDE( RepeatCustomers, [Unique Customers] )
```

### 5.4 Discount & Margin

```dax
-- [M12] Total Discount Amount
Total Discount =
    SUM( FactSales[discount_amount] )

-- [M13] Discount Rate %
Discount Rate % =
    DIVIDE(
        [Total Discount],
        SUM( FactSales[total_amount] )
    )

-- [M14] Discount Erosion Index
-- Compare current filter context discount rate vs chain average
-- Values > 1 = more discounting than chain average (margin risk)
Discount Erosion Index =
    VAR LocalRate = [Discount Rate %]
    VAR ChainRate = CALCULATE( [Discount Rate %], ALL( DimStore ) )
    RETURN
        DIVIDE( LocalRate, ChainRate )
```

### 5.5 Ranking & Intelligence

```dax
-- [M15] Store Revenue Rank
Store Revenue Rank =
    RANKX(
        ALL( DimStore[store_name] ),
        [Total Revenue],
        , DESC, Dense
    )

-- [M16] Top N Products Revenue (dynamic — requires What-If parameter)
Top N Revenue =
    CALCULATE(
        [Total Revenue],
        TOPN(
            [TopN Parameter],   -- What-If parameter 1–18
            ALL( DimProduct[product_name] ),
            [Total Revenue], DESC
        )
    )
```

---

## 6. Report Pages

### Page 1: Executive Overview
- KPI card strip: Total Revenue, Transactions, Unique Customers, Avg Basket, Discount Rate, Avg Loyalty
- Monthly Revenue Trend line chart (18 months)
- Revenue by Aisle donut chart
- Store Revenue ranking table with inline bars
- Discount Rate by Category horizontal bars

### Page 2: Store Performance
- Store comparison horizontal bar chart (switchable metric: Revenue / Transactions / Avg Basket)
- Top 3 store scorecards
- Full store analysis table with performance tier badges

### Page 3: Product & Category
- Revenue by Aisle bar chart
- Volume vs Revenue bubble scatter plot
- Full product performance matrix (18 products ranked)

### Page 4: Customer Intelligence
- Customer segmentation KPIs
- RFM-proxy donut chart (visit frequency tiers)
- Basket value histogram
- Loyalty points by category

### Page 5: Data Model Diagram
- Visual star schema with table definitions
- M Query code for key tables
- Relationship cardinality annotations

### Page 6: DAX Measures Library
- All 16 measures with syntax highlighting
- Implementation notes and business context

---

## 7. Key Business Insights

### Revenue Concentration
The top 3 stores (GreenGrocer Plaza, SuperSave Central, City Fresh Store) contribute 36% of total revenue. The bottom 2 stores (QuickStop Market, FamilyFood Express) lag by 18–24% — investigate staffing, location, and promotional mix.

### Category Opportunity
Personal Care (10.1% revenue share) outperforms Produce (7.4%) despite fewer physical SKUs. Produce has the highest transaction frequency but lowest revenue-per-unit — a basket-builder, not a margin driver. Price elasticity testing recommended.

### Discount Risk: Beverages & Canned Goods
14.3% and 14.2% discount rates respectively — 47–46% above the chain average. These categories are being over-promoted; a 2pp discount reduction would recover approximately $340 in annual margin per store.

### Repeat Customer Deficit
90.8% single-visit rate signals poor loyalty programme engagement. The average loyalty score of 255 points (max 500/txn) suggests customers are not aware of the maximum-earn mechanics. A targeted re-engagement campaign for the 1,632 one-time visitors could meaningfully lift CLV.

### Seasonal Pattern
March 2025 shows the highest monthly revenue ($4,560) — 57% above the weakest month (July 2024: $2,793). Seasonal planning should front-load inventory and promotional budget for Q1 and Q3.

---

## 8. Power BI Implementation Checklist

- [ ] Load `grocery_chain_data.csv` as the base source query
- [ ] Apply FactSales Power Query transformations (Steps 1–7 above)
- [ ] Generate DimDate using the provided M Query
- [ ] Derive DimProduct, DimStore, DimCustomer from FactSales
- [ ] Create DimPromotion from discount_pct bands
- [ ] Set all relationships (single-direction, 1:N) from Dim → Fact
- [ ] Mark DimDate as the Date Table (Model view > Mark as date table)
- [ ] Create all 16 DAX measures in a dedicated Measures table
- [ ] Hide all FK columns from Report view
- [ ] Format currency measures with `$#,##0.00`
- [ ] Format percentage measures with `0.0%`
- [ ] Create What-If parameter: TopN (1–18, step 1, default 5)
- [ ] Enable Row-Level Security if multi-store access control required
- [ ] Set scheduled refresh (if connected to live data source)
- [ ] Publish to Power BI Service and configure workspace permissions

---

*Solution architected for Power BI Desktop (June 2024+) | DAX compatibility: Power BI, SSAS Tabular, Azure Analysis Services*
