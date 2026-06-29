# utkarshshahare_2511900__part1_data_cleaning

# Retail Sales Order Data Cleaning & Analysis

## Problem Summary

A retail company operates across multiple regions in India and records order-level sales data in separate systems — a billing platform, a logistics tracker, and a CRM. When these systems were merged into a single export, the resulting dataset carried the baggage of all three: dates in four different formats, customer names that were the same person written three different ways, discount fields that contained negative numbers and percentage strings, and entire groups of rows that were exact copies of each other.

The goal was to take that raw, multi-system export and turn it into a dataset that analysts could actually trust — one where a pivot table on "sales by region" would not silently include cancelled orders, where a profit margin calculation would not divide by a negative number, and where every anomaly was documented rather than quietly deleted or overwritten.

---

## Dataset Description

| Property | Value |
|---|---|
| Source file | `raw_orders.xlsx` |
| Raw row count | 932 orders |
| Final row count | 912 orders (20 confirmed exact duplicates removed) |
| Final column count | 40 columns (26 original + 14 added during cleaning) |
| Date range | January 2024 – October 2025 |
| Regions | East, North, South, West |
| Categories | Furniture, Office Supplies, Technology |
| Customer segments | Consumer, Corporate, Home Office, Small Business |
| Order statuses | Completed (604), Cancelled (145), Returned (163) |
| Payment statuses | Paid (686), Pending (86), Refunded (71), Failed (69) |

---

## Tools Used

- **Python 3** — all data cleaning, validation, and report generation
  - `pandas` — data loading, transformation, and aggregation
  - `openpyxl` — Excel file creation, formatting, and multi-sheet workbook assembly
  - `numpy` — numerical operations and null handling
  - `re` (regex) — date format detection, text normalisation
  - `datetime` — date parsing and arithmetic
- **Microsoft Excel** — review and manual inspection of outputs

---

## Data Cleaning Summary

### Task 2 — Text Standardisation
- Standardised all 10 text columns: trimmed spaces, collapsed multiple spaces, applied Title Case.
- Fixed ~154 inconsistent values (case, spacing, duplicates).
- Filled 26 missing `region` values using city-to-region mapping.
- Retained 22 unresolved `ship_mode` nulls.

### Task 3 — Date Cleaning
- Standardised `order_date` and `ship_date` from 4 formats to `YYYY-MM-DD` using regex-based detection.
- Added `_std` date columns while preserving originals.
- Flagged 21 records where `ship_date < order_date` (not corrected).
- Created `shipping_delay_days`.

### Task 4 — Duplicate Handling
- Removed 20 duplicate rows from 40 exact duplicates (kept first occurrence).
- Flagged 24 conflicting duplicate `order_id` records (`sales`/`order_status`) for manual review.
- Logged all removed rows in `data_quality_report.xlsx`.

### Task 5 — Business Rules
- Applied 9 validation rules.
- Excluded `Cancelled`/`Failed Payment` orders from sales (`contributes_to_sales`); separated `Returned` orders.
- Flagged invalid discounts (negative/text values).
- If `quantity × unit_price × (1 − discount)` differed from `sales` by >±$0.02, set `discount_clean = 0` and flagged.
- Preserved all original values; added new columns.

### Task 6 — Calculated Columns
- Added: `cleaned_discount`, `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, `data_quality_flag`.
- Left 23 unresolved discount records as `NULL` (no estimation).
- `data_quality_flag`: Clean (814), Warning (32), Invalid (66).

### Tasks 7–8 — Validation & Reports
- Performed final consistency validation (no new issues).
- Generated 6 pivot tables in `pivot_summary.xlsx`:
  - Sales by region & category
  - Order count by ship mode
  - Profit margin by segment
  - Exception orders by region
  - Monthly sales trends
---

## Business Rules Applied

| Rule | Effect |
|---|---|
| Missing region → "Unknown" | Applied; 0 nulls remained after Task 2 region fill |
| Missing ship_mode → "Unknown" | 21 nulls set to "Unknown" in `ship_mode_clean` |
| Null discount → 0 if qty/price/sales all valid | 4 rows set to 0 in `discount_clean` |
| Discount must be 0–1; outside range = invalid | 23 rows flagged (15 negative, 8 text format) |
| Sales formula mismatch → set discount to 0, flag | 41 rows flagged; `discount_clean` zeroed |
| Cancelled orders excluded from sales summary | 145 rows marked `NO - Cancelled` |
| Failed payment excluded from sales summary | 37 rows marked `NO - Failed Payment` |
| Returned orders summarised separately | 128 rows marked `REFUND_SUMMARY` |
| Ship before order date → flag as invalid | 21 rows flagged in `shipping_record_flag` |

---

## Summary of Data Quality Issues

| Issue | Count | Status |
|---|---|---|
| Exact duplicate rows | 40 (20 pairs) | 20 removed; 20 first-occurrences retained |
| Conflicting duplicate order_ids | 24 rows (12 pairs) | Flagged; not deleted |
| Ship date before order date | 21 rows | Flagged; not corrected |
| Invalid discount (negative or text) | 23 rows | Flagged; calculated_sales = NULL |
| Sales formula mismatch | 41 rows | discount_clean set to 0; flagged |
| Missing ship_mode | 21 rows | Filled with "Unknown" |
| Missing region (pre-Task 2) | 26 rows | Filled via city-to-region lookup |
| Cancelled orders | 145 rows | Excluded from sales aggregations |
| Failed payment orders | 69 rows | Excluded from sales aggregations |
| Returned orders | 163 rows | Segmented separately |
| Overall data quality (Clean) | 814 rows | 89.3% of final dataset |

---

## Summary of Pivot Reports

All pivot tables use `calculated_sales` and `calculated_profit` (the validated, formula-derived figures), not the raw `sales` and `profit` columns. Cancelled and Failed Payment orders are excluded from all revenue metrics unless explicitly noted.

**PT1 — Sales & Profit by Region**  
South leads on total revenue (₹15.9L), followed closely by West (₹15.0L). East achieves the highest profit margin at 30.3%.

**PT2 — Sales & Profit by Category / Sub-Category**  
Technology is the top-performing category on both revenue (₹20.8L) and margin (30.3%). Within sub-categories, Machines stands out with a 34.6% margin despite mid-range revenue. Copiers is the single largest revenue sub-category.

**PT3 — Order Count by Ship Mode**  
All four ship modes are remarkably even: Standard Class (26.5%), First Class (24.5%), Second Class (24.3%), Same Day (22.4%). Twenty-one orders have no ship mode recorded.

**PT4 — Profit Margin by Segment**  
Home Office leads at 30.2%, Consumer at 30.0%, Corporate at 29.4%, and Small Business at 28.0%. The spread across segments is narrow at 2.1 percentage points.

**PT5 — Exception Orders by Region**  
North has the most exception orders (100), made up predominantly of Cancelled orders (48). In total, 310 of 912 orders (34.0%) are exceptions — a significant rate that warrants investigation into cancellation and returns drivers.

**PT6 — Monthly Sales Trend**  
June 2024 and February 2025 are the strongest revenue months. The 2024 full year achieved ₹29.8L in completed sales. October 2025 data is partial (10 orders only) and should not be used for period comparisons.

---
```markdown
## Key Business Insights

1. **34% exception rate** — 145 cancellations + 69 failed payments + 163 returns / 912 orders. North has 48 cancellations alone.
2. **Technology leads** — ₹20.8L revenue, 30.3% margin. Machines sub-category tops all at 34.6%.
3. **Discount integrity issues** — 41 orders fail sales formula; 23 have invalid discounts. ~7% of orders unverifiable.
4. **Regional margins tight but uneven** — East (30.3%) vs South (28.8%) = 1.5pp gap; may reflect product mix or discount policy.
5. **Shipping data errors** — 21 orders: ship date before order date; 21 orders: no ship mode recorded.

---

## Assumptions & Limitations

**Assumptions**
- Sales formula: `qty × unit_price × (1 − discount) = sales` with ±$0.02 rounding tolerance.
- `DD-MM-YYYY` assumed over `MM-DD-YYYY` (validated by day values > 12 in first segment).
- Region nulls filled from existing data patterns, not an external geographic reference.
- Negative discounts flagged only — may be surcharges or sign errors; source system must confirm.

**Limitations**
- 21 `ship_mode` nulls, 21 invalid ship dates, 24 conflicting duplicate `order_id`s unresolvable without source system access.
- 23 rows have NULL `calculated_sales`; excluded from all revenue/margin aggregations.
- Oct 2025 has only 10 completed orders (partial period) — do not compare YTD 2025 to full-year 2024.
- No PII anonymisation applied to `customer_name` or `customer_id`.
```
---

## Screenshots Included

| File | Contents |
|---|---|
| `raw_data_preview.png` | First rows of the raw dataset showing original mixed formats and issues |
| `cleaned_data_preview.png` | Final cleaned dataset with calculated columns and flag columns visible |
| `pivot_summary_1.png` | PT1 — Sales & Profit by Region pivot table |
| `pivot_summary_2.png` | PT4 — Profit Margin by Customer Segment pivot table |

---

## File Inventory

| File | Description |
|---|---|
| `raw_orders.xlsx` | Original source file — never modified |
| `cleaned_orders_final.xlsx` | Complete cleaned dataset (912 rows, 40 cols, 8 audit sheets) |
| `pivot_summary.xlsx` | 6 pivot tables + dashboard (7 sheets) |
| `data_quality_report.xlsx` | Duplicate analysis QA workbook (5 sheets) |
| `cleaning_log.md` | Chronological record of all cleaning actions |
| `README.md` | This file |
| `raw_data_preview.png` | Screenshot of raw data |
| `cleaned_data_preview.png` | Screenshot of cleaned data |
| `pivot_summary_1.png` | Screenshot: Sales & Profit by Region |
| `pivot_summary_2.png` | Screenshot: Profit Margin by Segment |
