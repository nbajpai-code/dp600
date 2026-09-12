# 🚀 Optimize Enterprise-Scale Semantic Models

> **DP-600 Exam Domain:** Implement and Manage Semantic Models (25–30%)  
> **Skills Measured (as of July 21, 2026):** Optimize enterprise-scale semantic models

---

## 📋 Table of Contents

- [Exam Objectives Breakdown](#-exam-objectives-breakdown)
- [1. Implement Performance Improvements](#1-implement-performance-improvements-in-queries-and-report-visuals)
- [2. Improve DAX Performance](#2-improve-dax-performance)
- [3. Configure Direct Lake](#3-configure-direct-lake)
- [4. Direct Lake: OneLake vs SQL Analytics Endpoint](#4-choose-between-direct-lake-on-onelake-and-direct-lake-on-sql-analytics-endpoint)
- [5. Implement Incremental Refresh](#5-implement-incremental-refresh-for-semantic-models)
- [Key Tools & Diagnostics](#-key-tools--diagnostics)
- [Practice Questions](#-practice-questions)
- [Resources & References](#-resources--references)

---

## 📌 Exam Objectives Breakdown

| # | Objective | Complexity |
|---|-----------|------------|
| 1 | Implement performance improvements in queries and report visuals | ⭐⭐⭐ |
| 2 | Improve DAX performance | ⭐⭐⭐⭐ |
| 3 | Configure Direct Lake, including default fallback and refresh behavior | ⭐⭐⭐⭐ |
| 4 | Choose between Direct Lake on OneLake and Direct Lake on SQL analytics endpoint | ⭐⭐⭐ |
| 5 | Implement incremental refresh for semantic models | ⭐⭐⭐⭐ |

---

## 1. Implement Performance Improvements in Queries and Report Visuals

### 🔑 Core Concepts

Performance bottlenecks in Power BI / Fabric semantic models typically fall into three layers:

```
Storage Engine (SE) → Formula Engine (FE) → Visual Rendering
```

- **Storage Engine (SE):** Columnar store (VertiPaq/Direct Lake). Highly parallelized. Fast.
- **Formula Engine (FE):** Single-threaded. Interprets DAX. Slow when overloaded.
- **Goal:** Push as much work to SE as possible.

### 🛠️ Query Performance Techniques

#### Reduce Visual Complexity
- Limit visuals per report page to **< 15–20 visuals**
- Use **aggregations** in the semantic model to pre-summarize large tables
- Avoid visual-level filters on high-cardinality columns — use slicers

#### Use Aggregations
Aggregations allow pre-computed summaries to speed up queries against large fact tables.

```
-- Detail table: Sales (billions of rows)
-- Agg table: Sales_Agg (grouped by Year, Month, Product)
-- Power BI automatically routes queries to agg table when possible
```

**Steps to configure:**
1. Import or create an aggregation table in Power Query
2. Set storage mode of agg table to **Import**
3. Set detail table to **Direct Lake** or **DirectQuery**
4. In the semantic model, right-click the agg table → **Manage aggregations**
5. Map each agg column to the detail table column

#### Optimize Data Model Structure
- Use **star schema** — avoid snowflake schemas in semantic models
- Remove unused columns (each column = memory overhead)
- Choose correct data types (e.g., `Integer` over `Decimal` where possible)
- Use **sorted columns** to enable dictionary compression

#### Reduce Cardinality

| Technique | Impact |
|-----------|--------|
| Bin high-cardinality numeric columns | Fewer dictionary entries |
| Replace text with integer keys | Smaller column size |
| Hide unused columns | Reduces model complexity |
| Use `Date` table with integer date key | Faster relationships |

#### Enable Query Reduction Settings (Report-level)
In Power BI Desktop → **Options → Query Reduction:**
- Add an **Apply button** to slicers
- Reduce queries sent on slicer interaction

### 🔍 Diagnostic Tools

| Tool | Purpose |
|------|---------|
| **Performance Analyzer** (Power BI Desktop) | Captures DAX query time, visual rendering time per visual |
| **DAX Studio** | Profile and analyze DAX queries, identify SE vs FE bottlenecks |
| **Tabular Editor** | Best Practice Analyzer, model inspection |
| **Fabric Capacity Metrics App** | Monitor capacity-level performance in Fabric |

---

## 2. Improve DAX Performance

### 🔑 The Formula Engine vs Storage Engine Distinction

```
Fast path:  DAX query → Storage Engine cache → Result ✅
Slow path:  DAX query → Formula Engine (complex iteration) → SE → Result ⚠️
```

**Key rule:** Anything the Storage Engine can handle in a single scan is fast. Complex row-by-row iteration forces FE involvement = slow.

### 📐 DAX Performance Best Practices

#### Use Variables (`VAR`)
Variables evaluate once and cache the result. Avoid re-evaluation.

```dax
-- ❌ Slow: CALCULATE evaluated multiple times
Slow Measure =
DIVIDE(
    CALCULATE(SUM(Sales[Amount]), Sales[Year] = 2024),
    CALCULATE(SUM(Sales[Amount]), Sales[Year] = 2023)
)

-- ✅ Fast: Variables cache results
Fast Measure =
VAR Sales2024 = CALCULATE(SUM(Sales[Amount]), Sales[Year] = 2024)
VAR Sales2023 = CALCULATE(SUM(Sales[Amount]), Sales[Year] = 2023)
RETURN DIVIDE(Sales2024, Sales2023)
```

#### Prefer `SUMX` / Iterator Patterns Carefully
Iterators (`SUMX`, `AVERAGEX`, etc.) operate row-by-row in the FE. Use sparingly.

```dax
-- ❌ Unnecessary iterator
Bad = SUMX(Sales, Sales[Qty] * Sales[Price])

-- ✅ Better: Pre-calculate at source or use a calculated column if static
Better = SUM(Sales[ExtendedAmount])  -- where ExtendedAmount = Qty * Price is a column
```

#### Avoid `IF` for Large Tables
`IF` evaluated on a virtual table iterates row-by-row:

```dax
-- ❌ Slow: Row-by-row IF evaluation
Slow = SUMX(Sales, IF(Sales[Amount] > 1000, Sales[Amount], 0))

-- ✅ Fast: Filter first, then aggregate
Fast = CALCULATE(SUM(Sales[Amount]), Sales[Amount] > 1000)
```

#### Table Functions: Performance Considerations

| Function | Performance Note |
|----------|-----------------|
| `FILTER` | Can be slow on large tables — prefer column filters in CALCULATE |
| `ALL` | Fast — removes all filters |
| `ALLEXCEPT` | Efficient for partial filter removal |
| `VALUES` | Returns distinct values — fast |
| `SUMMARIZE` | Can be slow; prefer `SUMMARIZECOLUMNS` for queries |

#### Windowing Functions (DAX)

```dax
-- WINDOW function for running totals
Running Total =
CALCULATE(
    SUM(Sales[Amount]),
    WINDOW(1, ABS, 0, REL, ORDERBY(Sales[Date]), PARTITIONBY(Sales[Product]))
)

-- INDEX for relative row access
Prev Month Sales =
CALCULATE(
    SUM(Sales[Amount]),
    INDEX(-1, ORDERBY('Date'[Month]))
)
```

### 🧪 Using DAX Studio for Profiling

1. Connect DAX Studio to your semantic model (Power BI Desktop or XMLA endpoint)
2. Run a query → click **Server Timings**
3. Analyze:
   - **SE Queries:** Fast parallel queries — aim to maximize
   - **FE Duration:** Should be minimal — high FE = optimization needed
   - **Cache hits:** Repeated queries benefit from SE cache

```dax
-- DAX Studio example: profile a measure
EVALUATE
SUMMARIZECOLUMNS(
    'Date'[Year],
    "Sales", [Total Sales]
)
```

---

## 3. Configure Direct Lake

### 🔑 What is Direct Lake?

Direct Lake is a **Fabric-native storage mode** for semantic models that queries Delta Parquet files directly from OneLake without importing data into memory — combining the **freshness of DirectQuery** with the **speed of Import mode**.

```
Traditional Import:  Data → Copy into VertiPaq memory → Query
DirectQuery:         Query → Source database (slow, every query)
Direct Lake:         Data in OneLake Delta → Load frames on demand → Query ⚡
```

### ⚙️ Direct Lake Configuration

#### Framing
- When a Direct Lake model is queried, Fabric **frames** the Delta table (reads metadata, builds column dictionaries)
- Frames are cached in memory — subsequent queries are fast
- Frames are invalidated when the underlying Delta table changes (triggers re-framing)

#### Default Fallback Behavior
Direct Lake can **fall back to DirectQuery** when:
- A query uses an unsupported feature
- A table column exceeds framing limits (row count or column count thresholds)
- Relationships cross different storage modes

**Configure fallback in the semantic model settings:**
1. Open semantic model in Fabric workspace
2. Go to **Settings → Direct Lake**
3. Choose fallback behavior:
   - **Allowed** (default): Falls back to DirectQuery automatically
   - **Not Allowed**: Returns an error if DirectQuery fallback required — best for performance SLAs

#### Framing Limits (as of 2026)

| Resource | Limit |
|----------|-------|
| Columns per table | 20,000 |
| Rows per table (framing) | Governed by F-SKU capacity |
| Tables per model | Governed by memory |

#### Refresh Behavior
Direct Lake models don't need scheduled refresh (data is live in OneLake). However:
- **Framing refresh** occurs automatically when Delta table is updated
- For **manual framing refresh**: Use the Fabric REST API or XMLA endpoint
- **Calculated columns/tables** still need refresh (they use Import mode internally)

```python
# Refresh a Direct Lake semantic model via REST API
import requests

url = "https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/semanticmodels/{modelId}/refresh"
headers = {"Authorization": "Bearer {token}"}
body = {"type": "full"}

response = requests.post(url, headers=headers, json=body)
```

---

## 4. Choose Between Direct Lake on OneLake and Direct Lake on SQL Analytics Endpoint

### 📊 Comparison Table

| Feature | Direct Lake on **OneLake** | Direct Lake on **SQL Analytics Endpoint** |
|---------|---------------------------|------------------------------------------|
| **Data source** | Delta Parquet files in OneLake | SQL Analytics Endpoint (Lakehouse/Warehouse) |
| **Freshness** | Real-time as Delta files update | Near-real-time (SQL endpoint refresh latency) |
| **Transformations** | None — raw Delta tables | SQL views, computed columns, stored procedures |
| **Security** | OneLake-level (ADLS access) | SQL endpoint-level (row/column security) |
| **Performance** | Fastest (direct file access) | Slightly slower (SQL endpoint layer) |
| **Use case** | Fresh raw analytics data | Transformed/governed data with SQL security |

### 🎯 Decision Guide

```
Need maximum performance and freshness?
    → Direct Lake on OneLake

Need row/column-level security defined at SQL layer?
    → Direct Lake on SQL Analytics Endpoint

Need transformations (views, computed columns)?
    → Direct Lake on SQL Analytics Endpoint

Need to query directly from lakehouse Delta tables?
    → Direct Lake on OneLake
```

---

## 5. Implement Incremental Refresh for Semantic Models

### 🔑 What is Incremental Refresh?

Incremental refresh partitions data into historical (frozen) and current (refreshed) slices, so only recent data is re-loaded during refresh.

```
Without Incremental Refresh: Refresh ALL rows every time (slow, resource-heavy)
With Incremental Refresh:    Refresh only recent window (fast, efficient)
```

### ⚙️ Configuration Steps

#### Step 1: Define RangeStart and RangeEnd Parameters in Power Query
These **exact** parameter names are required by Fabric:

```m
// In Power Query Editor → Manage Parameters
// Create two parameters:
RangeStart  = DateTime.FromText("2024-01-01T00:00:00")  // Type: Date/Time
RangeEnd    = DateTime.FromText("2024-12-31T23:59:59")  // Type: Date/Time
```

#### Step 2: Filter the Date Column in Power Query

```m
= Table.SelectRows(#"Changed Type", each
    [OrderDate] >= RangeStart and [OrderDate] < RangeEnd
)
```

> ⚠️ Use `>=` and `<` (not `<=`) to avoid data overlap between partitions.

#### Step 3: Configure Incremental Refresh Policy (Power BI Desktop)

1. Right-click the table in the Fields pane → **Incremental refresh**
2. Configure:
   - **Archive data starting:** e.g., 5 years back (historical range)
   - **Incrementally refresh data starting:** e.g., 30 days back (rolling window)
   - **Detect data changes:** (Optional) Use a `Last Modified` timestamp column
   - **Only refresh complete periods:** Prevents incomplete day/month partitions

#### Step 4: Publish to Fabric (Premium/Fabric capacity required)

After publishing:
- Partitions are auto-created based on the policy
- Refresh runs only the incremental window

### 📊 Partition Granularity

| Archive Period | Granularity |
|---------------|-------------|
| > 1 year | Yearly partitions |
| > 1 month | Monthly partitions |
| > 1 day | Daily partitions |
| Current window | Hourly (if enabled) |

### 🔄 Real-Time Data with Hybrid Tables

Hybrid tables combine incremental refresh (Import mode) with DirectQuery for the current partition:

```
Historical data (2020-2024) → Import mode partitions (fast)
Current period (last 30 days) → DirectQuery partition (real-time)
```

**Configure:**
1. In incremental refresh settings → Enable **"Get the latest data in real time with DirectQuery"**
2. This creates a DirectQuery partition for the current period
3. Requires Premium or Fabric capacity

### ⚠️ Common Pitfalls

| Issue | Solution |
|-------|---------|
| Parameters not named exactly `RangeStart`/`RangeEnd` | Rename them — Fabric requires exact names |
| Filter uses `<=` instead of `<` for end | Data overlap between partitions |
| Source doesn't support query folding | Incremental refresh won't work — full table loaded each time |
| Publishing to shared capacity | Incremental refresh requires Premium or Fabric capacity |

---

## 🔧 Key Tools & Diagnostics

| Tool | Use Case | Access |
|------|---------|--------|
| **Performance Analyzer** | Capture per-visual query times | Power BI Desktop → View → Performance Analyzer |
| **DAX Studio** | Profile SE/FE, optimize queries | [daxstudio.org](https://daxstudio.org/) |
| **Tabular Editor 3** | Best Practice Analyzer, partition management | [tabulareditor.com](https://tabulareditor.com/) |
| **Fabric Capacity Metrics App** | Workspace-level capacity monitoring | Install from AppSource |
| **XMLA Endpoint** | Advanced model management, partition inspection | Connect via SSMS or Tabular Editor |
| **VertiPaq Analyzer** | Column size, cardinality analysis | Available in DAX Studio |

---

## ❓ Practice Questions

**Q1:** You have a semantic model with a 500M-row fact table. Reports are slow. Which technique provides the BEST performance improvement?
- A) Convert the fact table to DirectQuery
- B) Create an aggregation table in Import mode  ✅
- C) Enable bidirectional relationships
- D) Add more columns to the model

> **Explanation:** Aggregations pre-summarize data, routing most queries to the small agg table without touching the full fact table.

---

**Q2:** A DAX measure uses `SUMX` over a 10M-row table. Performance is poor. What is the BEST optimization?
- A) Replace `SUMX` with `SUM` using a pre-calculated column  ✅
- B) Add an index to the table
- C) Enable Direct Lake mode
- D) Use `KEEPFILTERS`

> **Explanation:** Moving the calculation to a stored column eliminates row-by-row FE iteration at query time.

---

**Q3:** Your Direct Lake semantic model falls back to DirectQuery for some visuals. You want to prevent this and return an error instead. What should you configure?
- A) Disable automatic refresh
- B) Set fallback to "Not Allowed" in Direct Lake settings  ✅
- C) Switch to Import mode
- D) Remove calculated columns

---

**Q4:** Which parameter names are REQUIRED for incremental refresh in Fabric?
- A) `StartDate` and `EndDate`
- B) `DateStart` and `DateEnd`
- C) `RangeStart` and `RangeEnd`  ✅
- D) `FilterStart` and `FilterEnd`

---

**Q5:** You need a semantic model where the last 7 days show real-time data and historical data is in Import mode. What should you use?
- A) DirectQuery mode
- B) Import mode with scheduled refresh
- C) Hybrid tables with incremental refresh  ✅
- D) Direct Lake on SQL Analytics Endpoint

---

## 📚 Resources & References

| Resource | Link |
|----------|------|
| Direct Lake overview | [learn.microsoft.com/fabric/fundamentals/direct-lake-overview](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview) |
| Incremental refresh | [learn.microsoft.com/power-bi/connect-data/incremental-refresh-overview](https://learn.microsoft.com/power-bi/connect-data/incremental-refresh-overview) |
| DAX optimization guide | [learn.microsoft.com/dax/best-practices/dax-variables](https://learn.microsoft.com/dax/best-practices/dax-variables) |
| Aggregations in Power BI | [learn.microsoft.com/power-bi/transform-model/aggregations-advanced](https://learn.microsoft.com/power-bi/transform-model/aggregations-advanced) |
| Direct Lake fallback | [learn.microsoft.com/fabric/fundamentals/direct-lake-analyze-query-processing](https://learn.microsoft.com/fabric/fundamentals/direct-lake-analyze-query-processing) |
| Performance Analyzer | [learn.microsoft.com/power-bi/create-reports/desktop-performance-analyzer](https://learn.microsoft.com/power-bi/create-reports/desktop-performance-analyzer) |
| Hybrid tables | [learn.microsoft.com/power-bi/connect-data/service-premium-incremental-refresh](https://learn.microsoft.com/power-bi/connect-data/service-premium-incremental-refresh) |
| DAX Studio | [daxstudio.org](https://daxstudio.org/) |
| Tabular Editor | [tabulareditor.com](https://tabulareditor.com/) |

---

*Last updated: September 2026 | DP-600 Skills measured as of July 21, 2026*
