# 🛒 Blinkit Intelligent Sales Analytics Dashboard

> **A Professional-Grade Power BI Business Intelligence Solution for Blinkit Supershop Sales Data**

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Data Model](https://img.shields.io/badge/Star%20Schema-Normalized-brightgreen?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production%20Ready-success?style=for-the-badge)

---

## 📌 Project Overview

This repository contains a fully normalized, enterprise-grade **Power BI semantic model** built on the BlinkIT Grocery Retail dataset. It transforms a raw flat-file transaction table into a scalable star-schema data model with advanced DAX intelligence, ready for executive reporting, strategic decision-making, and operational performance monitoring.

**Project Title:** Blinkit Intelligent Sales Analytics Dashboard
**Source Table:** `Supershop Sales Data` (master fact source)
**Model Type:** Tabular — Compatibility Level 1600

---

## 🗂️ Repository Structure

```
📦 blinkit-sales-analytics/
├── 📄 README.md
├── 📊 BlinkIT Grocery Sales.pbix          # Power BI Desktop file
├── 📁 screenshots/
│   ├── data-model.png                     # Star schema diagram
│   ├── executive-dashboard.png            # Main KPI overview page
│   ├── product-analysis.png               # Product performance page
│   ├── outlet-analysis.png                # Outlet performance page
│   └── delivery-analytics.png            # Delivery & operations page
└── 📁 docs/
    └── dax-measures-reference.md          # Full DAX measure documentation
```

---

## 🏗️ Data Model Architecture

The model follows a strict **star schema** with a single fact table surrounded by dedicated dimension tables — all derived from the master `Supershop Sales Data` source using calculated DAX expressions.

```
                    ┌──────────────┐
                    │  Dim_Date    │ ← Marked as Date Table
                    │  (Calendar)  │
                    └──────┬───────┘
                           │
  ┌──────────────┐         │         ┌──────────────┐
  │ Dim_Customer │         │         │  Dim_Product  │
  │              │         │         │              │
  └──────┬───────┘         │         └──────┬───────┘
         │                 │                │
         └────────── ┌─────▼──────┐ ────────┘
                     │  Supershop │
                     │ Sales Data │  ◄── Master Fact Table
                     │  (FACT)    │
         ┌────────── └─────┬──────┘ ────────┐
         │                 │                │
  ┌──────▼───────┐         │         ┌──────▼──────────────┐
  │  Dim_Outlet  │         │         │ Dim_DeliveryStatus  │
  │              │         │         │                     │
  └──────────────┘         │         └─────────────────────┘
                    ┌──────▼───────┐
                    │  _Measures   │
                    │ (43 DAX KPIs)│
                    └──────────────┘
```

### Tables at a Glance

| Table | Type | Row Source | Description |
|---|---|---|---|
| `Supershop Sales Data` | **Fact** | Raw data import | All transactions — orders, products, outlets, delivery |
| `Dim_Customer` | Calculated Dim | DISTINCT on Fact | Unique customer profiles |
| `Dim_Product` | Calculated Dim | DISTINCT on Fact | Product catalogue with attributes |
| `Dim_Outlet` | Calculated Dim | DISTINCT on Fact | Store profiles and classifications |
| `Dim_Date` | Calculated Date Table | CALENDAR from Fact | Full date spine with time attributes |
| `Dim_DeliveryStatus` | Calculated Dim | DISTINCT on Fact | Delivery status lookup |
| `_Measures` | Measures Table | N/A | 43 centralized DAX measures |

---

## 📐 Source Schema — `Supershop Sales Data`

| Column | Data Type | Description |
|---|---|---|
| `DataSource` | String | Source system identifier |
| `CustomerID` | String | Unique customer key |
| `Name` | String | Customer full name |
| `Phone` | String | Customer contact number |
| `Address` | String | Delivery address |
| `City` | String | Customer city |
| `OrderID` | String | Order header identifier |
| `OrderDateTime` | DateTime | Date and time of order placement |
| `DeliveryDateTime` | DateTime | Date and time of delivery |
| `DeliveryStatus` | String | Current delivery status |
| `OrderDetailsID` | String | Order line identifier (PK) |
| `Item Identifier` | String | Product SKU |
| `ProductName` | String | Product display name |
| `Quantity` | Int64 | Units ordered |
| `PricePerUnit` | Int64 | Unit selling price |
| `TotalAmount` | Int64 | Line total (Quantity × PricePerUnit) |
| `Item Fat Content` | String | Fat content category (Regular / Low Fat) |
| `Item Type` | String | Product category |
| `Item Visibility` | Double | Shelf visibility score |
| `Item Weight` | Double | Product weight |
| `Outlet Identifier` | String | Store identifier |
| `Outlet Establishment Year` | Int64 | Year outlet was established |
| `Outlet Location Type` | String | City tier (Tier 1 / Tier 2 / Tier 3) |
| `Outlet Size` | String | Outlet size classification |
| `Outlet Type` | String | Store format type |
| `Rating` | Double | Customer satisfaction rating (1–5) |

---

## 📊 DAX Measures Reference — 43 KPIs Across 6 Folders

### 💰 Sales KPIs (8 Measures)

| Measure | Formula Logic | Format |
|---|---|---|
| `Total Sales` | `SUM(TotalAmount)` | `#,##0.00 ৳` |
| `Total Orders` | `COUNTROWS(Fact)` | `#,##0` |
| `Total Quantity` | `SUM(Quantity)` | `#,##0` |
| `Avg Order Value` | `Total Sales ÷ Total Orders` | `#,##0.00 ৳` |
| `Avg Price Per Unit` | `AVERAGE(PricePerUnit)` | `#,##0.00 ৳` |
| `Unique Customers` | `DISTINCTCOUNT(CustomerID)` | `#,##0` |
| `Unique Products` | `DISTINCTCOUNT(Item Identifier)` | `#,##0` |
| `Active Outlets` | `DISTINCTCOUNT(Outlet Identifier)` | `#,##0` |

### ⭐ Rating & Quality (4 Measures)

| Measure | Formula Logic | Format |
|---|---|---|
| `Avg Rating` | `AVERAGE(Rating)` | `0.00` |
| `High Rated Orders` | Orders with Rating ≥ 4 | `#,##0` |
| `High Rating %` | `High Rated Orders ÷ Total Orders` | `0.0%` |
| `Revenue-Weighted Rating` | Σ(Rating × Amount) ÷ Total Sales | `0.00` |

### 🚚 Delivery & Operations (5 Measures)

| Measure | Formula Logic | Format |
|---|---|---|
| `Delivered Orders` | Orders where Status = "Delivered" | `#,##0` |
| `Delivery Success Rate` | `Delivered Orders ÷ Total Orders` | `0.0%` |
| `Pending Orders` | Orders where Status ≠ "Delivered" | `#,##0` |
| `Avg Delivery Time (Hrs)` | `AVERAGEX(DATEDIFF(Order, Delivery, HOUR))` | `0.0 hrs` |
| `Delivered Revenue` | Sales filtered to Delivered only | `#,##0.00 ৳` |

### 📅 Time Intelligence (8 Measures)

| Measure | Formula Logic | Format |
|---|---|---|
| `Sales LY` | `SAMEPERIODLASTYEAR` | `#,##0.00 ৳` |
| `Sales YoY Growth %` | `(Sales - Sales LY) ÷ Sales LY` | `+0.0%;-0.0%` |
| `Sales YTD` | `DATESYTD` | `#,##0.00 ৳` |
| `Sales QTD` | `DATESQTD` | `#,##0.00 ৳` |
| `Sales MTD` | `DATESMTD` | `#,##0.00 ৳` |
| `Sales Prev Month` | `PREVIOUSMONTH` | `#,##0.00 ৳` |
| `Sales MoM Growth %` | `(Sales - Prev Month) ÷ Prev Month` | `+0.0%;-0.0%` |
| `Sales 3M Rolling Avg` | `AVERAGEX(DATESINPERIOD(-3M))` | `#,##0.00 ৳` |

### 🏪 Product & Outlet Analytics (8 Measures)

| Measure | Formula Logic |
|---|---|
| `Sales - Regular Fat` | Revenue filtered to Regular fat items |
| `Sales - Low Fat` | Revenue filtered to Low Fat items |
| `Product Sales Share %` | `Sales ÷ ALL(Dim_Product)` |
| `Sales - Supermarket T1` | Revenue from Supermarket Type1 outlets |
| `Sales - Grocery Store` | Revenue from Grocery Store outlets |
| `Outlet Sales Share %` | `Sales ÷ ALL(Dim_Outlet)` |
| `Avg Sales Per Outlet` | `Total Sales ÷ Active Outlets` |
| `Avg Sales Per Customer` | `Total Sales ÷ Unique Customers` |

### 🌍 Geography & Tier (4 Measures)

| Measure | Formula Logic |
|---|---|
| `Sales - Tier 1` | Revenue from Tier 1 city outlets |
| `Sales - Tier 2` | Revenue from Tier 2 city outlets |
| `Sales - Tier 3` | Revenue from Tier 3 city outlets |
| `City Sales Share %` | `Sales ÷ ALL(Outlet Location Type)` |

### 🏆 Rankings & Intelligence (6 Measures)

| Measure | Formula Logic |
|---|---|
| `Product Sales Rank` | `RANKX(ALL Products, Total Sales, DESC)` |
| `Outlet Sales Rank` | `RANKX(ALL Outlets, Total Sales, DESC)` |
| `Cumulative Sales %` | Pareto/ABC cumulative % analysis |
| `Sales vs Avg Outlet` | Sales delta vs. average outlet performance |
| `Repeat Customers` | Customers with 2+ order lines |
| `Repeat Customer Rate` | `Repeat Customers ÷ Unique Customers` |

---

## 🔗 Relationships

| From Table | From Column | To Table | To Column | Cardinality | Active |
|---|---|---|---|---|---|
| Supershop Sales Data | `CustomerID` | Dim_Customer | `CustomerID` | Many → One | ✅ |
| Supershop Sales Data | `Item Identifier` | Dim_Product | `Item Identifier` | Many → One | ✅ |
| Supershop Sales Data | `Outlet Identifier` | Dim_Outlet | `Outlet Identifier` | Many → One | ✅ |
| Supershop Sales Data | `DeliveryStatus` | Dim_DeliveryStatus | `Delivery Status` | Many → One | ✅ |
| Supershop Sales Data | `OrderDateTime` | Dim_Date | `Date` | Many → One | ✅ |

All relationships use **single-direction cross-filtering** from dimension to fact for optimal performance and correct filter propagation.

---

## 🗂️ Drill-Down Hierarchies

### 📅 Date Hierarchy (`Dim_Date`)
```
Year → Quarter → Month → Day
```

### 🏪 Outlet Hierarchy (`Dim_Outlet`)
```
Location Tier → Outlet Type → Outlet
```

### 🛍️ Product Hierarchy (`Dim_Product`)
```
Category → Fat Content → Product
```

---

## 🚀 Getting Started

### Prerequisites
- **Power BI Desktop** (June 2024 or later recommended)
- Windows 10 / 11

### Setup Instructions

1. **Clone this repository**
   ```bash
   git clone https://github.com/your-username/blinkit-sales-analytics.git
   cd blinkit-sales-analytics
   ```

2. **Open the `.pbix` file** in Power BI Desktop

3. **Refresh the data**
   - Go to **Home → Refresh**
   - Ensure the source Excel/CSV file path is correctly configured under **Transform Data → Data Source Settings**

4. **Explore the model**
   - Navigate to **Model View** to see the star schema relationships
   - Open **_Measures** table to browse all 43 DAX KPIs by display folder

---

## 📋 Suggested Dashboard Pages

Use the measures and hierarchies to build these recommended report pages:

| Page | Key Visuals | Key Measures |
|---|---|---|
| **Executive Overview** | KPI Cards, Revenue Trend | Total Sales, YoY Growth %, Avg Order Value |
| **Sales Performance** | Line/Bar charts by Month/Quarter | Sales MTD, QTD, YTD, MoM Growth % |
| **Product Analysis** | Treemap, Pareto chart | Product Sales Rank, Cumulative Sales %, Product Sales Share % |
| **Outlet Intelligence** | Map, Matrix | Outlet Sales Rank, Sales - Tier 1/2/3, Avg Sales Per Outlet |
| **Customer Insights** | Donut, Scatter | Unique Customers, Repeat Rate, Avg Sales Per Customer |
| **Delivery Operations** | Gauge, Table | Delivery Success Rate, Avg Delivery Time, Pending Orders |
| **Rating Dashboard** | Star visual, Heatmap | Avg Rating, Revenue-Weighted Rating, High Rating % |

---

## 🛠️ Technical Design Decisions

- **Calculated dim tables** are derived via DAX `DISTINCT + SELECTCOLUMNS` from the fact table, ensuring zero ETL dependency and single-source-of-truth integrity.
- **`_Measures` table** uses a placeholder `DATATABLE` — all KPIs are housed here for clean model navigation.
- **`Dim_Date`** is built with `CALENDAR()` from the actual `MIN/MAX` of `OrderDateTime`, ensuring no orphan dates.
- **All relationships** are inactive-safe: active relationships use date-only joining on `OrderDateTime` → `Dim_Date[Date]`.
- **Display folders** on measures follow business domain grouping, not technical grouping, for end-user friendliness.

---

## 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you'd like to change.

1. Fork the project
2. Create your feature branch (`git checkout -b feature/new-measure`)
3. Commit your changes (`git commit -m 'Add: new customer lifetime value measure'`)
4. Push to the branch (`git push origin feature/new-measure`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

---

## 👤 Author

**Developed as part of a senior Power BI architecture engagement.**

- Built with Power BI Desktop + Power BI MCP Server
- Semantic model: Tabular (Compatibility Level 1600)
- DAX version: Power BI Desktop (latest)

---

> *"Data is the new oil — but only if it's refined."*
> This dashboard refines raw Blinkit transaction data into executive-ready intelligence.
