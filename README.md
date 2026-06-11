# 🪟 Window Functions — BigQuery SQL

BigQuery SQL project exploring window functions using Greenweez sales data (`gwz_orders_17` and `gwz_sales_17`).

---

## 🎯 Objective

Window functions allow calculations across rows without collapsing them into groups — unlike `GROUP BY`, all rows are preserved. This project covers the three core ranking functions and builds a real-world pipeline that tags each customer's first-ever purchase.

This project answers:

- How do `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` differ?
- When should each one be used?
- How can window functions be used to identify a customer's first order?
- How do you build a daily new customer acquisition report from raw sales data?

---

## 🗂️ Dataset

Two tables from BigQuery (`course17` dataset):

| Table | Rows | Description |
|---|---|---|
| `gwz_orders_17` | 142,409 | One row per order — date, customer, turnover, margin, costs |
| `gwz_sales_17` | 1,168,081 | One row per product line — order, product, category, turnover, purchase cost |

---

## 🪜 Project Steps

### 1. ROW_NUMBER() — Basic Ranking

Assigns a unique sequential number to each row within a partition.

```sql
SELECT *,
  ROW_NUMBER() OVER(
    PARTITION BY customers_id
  ) AS nr
FROM course17.gwz_orders_17;
```

→ Each customer's orders get numbered 1, 2, 3... regardless of ties.

---

### 2. ROW_NUMBER() vs RANK() — With ORDER BY

Adding `ORDER BY date_date` makes the ranking meaningful — earlier orders get lower numbers.

```sql
ROW_NUMBER() OVER(PARTITION BY customers_id ORDER BY date_date) AS rn,
RANK()       OVER(PARTITION BY customers_id ORDER BY date_date) AS rk
```

**Key difference on tied dates:**

| Function | Tied rows | Next row |
|---|---|---|
| `ROW_NUMBER()` | 1, 2 | 3 |
| `RANK()` | 1, 1 | 3 (skips 2) |

→ Use `ROW_NUMBER()` when you need exactly one "first" row per customer.

---

### 3. gwz_orders_17_rn — First Order Flag

Builds a table with `is_new` column: `1` for a customer's first order, `0` for all others.

```sql
CREATE OR REPLACE TABLE course17.gwz_orders_17_rn AS
WITH orders_with_rn AS (
  SELECT *, ROW_NUMBER() OVER(
    PARTITION BY customers_id ORDER BY date_date
  ) AS rn
  FROM course17.gwz_orders_17
)
SELECT *,
  CASE WHEN rn = 1 THEN 1 ELSE 0 END AS is_new
FROM orders_with_rn;
```

---

### 4. Applying to gwz_sales_17

Sales table has multiple product lines per order. `ORDER BY date_date, orders_id` ensures stable ranking even when a customer has multiple items in one order.

```sql
ROW_NUMBER() OVER(PARTITION BY customers_id ORDER BY date_date, orders_id) AS rn,
RANK()       OVER(PARTITION BY customers_id ORDER BY date_date, orders_id) AS rk
```

---

### 5. ROW_NUMBER() vs RANK() vs DENSE_RANK()

Full comparison on `gwz_sales_17` to see behavior when multiple product lines share the same `orders_id`:

| Function | Tied rows | Next row | Use case |
|---|---|---|---|
| `ROW_NUMBER()` | 1, 2, 3 | 4 | Unique first row |
| `RANK()` | 1, 1, 1 | 4 (skips) | Score-based ranking |
| `DENSE_RANK()` | 1, 1, 1 | 2 (no skip) | Order-level ranking |

**Why `DENSE_RANK()` for sales table?**
Multiple product lines belong to the same order. `DENSE_RANK()` gives the same rank to all lines in the same order — so `ds_rk = 1` correctly tags the entire first order, not just the first product line.

---

### 6. gwz_sales_17_ds_rk — Pipeline

Full pipeline from raw sales data to new customer analysis:

```
gwz_sales_17 (raw)
  ↓ DENSE_RANK() — assign order sequence per customer
  ↓ CASE WHEN ds_rk = 1 → is_new = 1
gwz_sales_17_ds_rk (analytical table)
  ↓ WHERE is_new = 1, GROUP BY date_date
Daily new customer report
```

**Final query — Daily new customer count:**

```sql
SELECT
  date_date,
  COUNT(*) AS new_customers
FROM course17.gwz_sales_17_ds_rk
WHERE is_new = 1
GROUP BY date_date;
```

→ 153 rows — one per day, showing how many new customers placed their first order.

---

## 📊 Key Findings

- `ROW_NUMBER()` is the standard tool for first-purchase detection and customer acquisition analysis
- `RANK()` skips numbers after ties — useful for leaderboard-style rankings
- `DENSE_RANK()` is the right choice when multiple rows share the same logical event (e.g. multiple products in one order)
- The `is_new` flag enables clean daily, weekly, or monthly new customer reporting from raw transaction data

---

## 🛠️ SQL Techniques Used

| Technique | Purpose |
|---|---|
| `ROW_NUMBER() OVER(PARTITION BY ... ORDER BY ...)` | Unique sequential ranking per customer |
| `RANK() OVER(...)` | Ranking with gaps on ties |
| `DENSE_RANK() OVER(...)` | Ranking without gaps on ties |
| `PARTITION BY` | Reset ranking for each customer |
| `ORDER BY` inside `OVER()` | Define ranking order |
| `CASE WHEN` | Convert rank to is_new flag |
| `WITH` (CTE) | Build intermediate ranked table |
| `CREATE OR REPLACE TABLE` | Save analytical tables in BigQuery |
| `WHERE is_new = 1` | Filter to first-order rows only |
| `GROUP BY date_date` | Daily aggregation |

---

## 💾 Output Tables

```
course17/
├── gwz_orders_17_rn     ← Orders with ROW_NUMBER and is_new flag
└── gwz_sales_17_ds_rk   ← Sales with DENSE_RANK and is_new flag
```

---

## 🔧 Tools

- **Google BigQuery** — SQL engine and data warehouse
- **SQL** — All analysis in standard SQL with BigQuery-specific window functions
