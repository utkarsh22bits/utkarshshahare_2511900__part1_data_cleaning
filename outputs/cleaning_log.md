# Data Cleaning Log
**Project:** Retail Sales Order Data Cleaning & Validation  
**Source File:** `raw_orders.xlsx` (932 rows, never modified)  
**Final Output:** `cleaned_orders.xlsx` (912 rows, 40 columns)  
**Tools Used:** Python 3 (pandas, openpyxl, numpy, re, datetime)  
**Tasks Completed:** 2 through 8  

---

## Task 2 — Standardize Text Fields

**Scope:** 10 text columns: `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`

### Issues Found
- Leading and trailing whitespace across all 10 columns
- Multiple internal spaces (e.g. `"Aarav  Sharma"`, `"Small  Business"`, `"Office  Supplies"`)
- Inconsistent casing: ALL-CAPS (`"PRIYA MENON"`, `"HOME OFFICE"`, `"FURNITURE"`), all-lowercase (`"mira das"`, `"consumer"`, `"binders"`), and mixed
- 26 null values in `region`
- 22 null values in `ship_mode`
- Trailing-space variants creating false unique values (e.g. `"Ananya Rao"` vs `"Ananya Rao "`)

### Cleaning Actions
1. Stripped all leading/trailing whitespace (`TRIM()` equivalent)
2. Collapsed multiple consecutive internal spaces to a single space via regex
3. Applied Title Case (`PROPER()` equivalent) to all 10 columns uniformly
4. Filled 26 null `region` values using a deterministic city-to-region lookup map built from non-null rows (`VLOOKUP()` equivalent)
5. Retained 22 null `ship_mode` values — cannot be inferred from available columns; flagged for source lookup

### Canonical Values Established
| Column | Canonical Values |
|---|---|
| segment | Consumer, Corporate, Home Office, Small Business |
| region | East, North, South, West |
| category | Furniture, Office Supplies, Technology |
| order_status | Cancelled, Completed, Returned |
| payment_status | Failed, Paid, Pending, Refunded |
| ship_mode | First Class, Same Day, Second Class, Standard Class (+ 22 nulls) |

### Assumptions Made
- City names treated as uniquely mappable to a single region. Two Rajasthan rows (Jaipur → North, Udaipur → West) inferred from the majority pattern; flagged for source verification.
- Title Case applied uniformly; no domain-specific abbreviation exceptions identified.

### Records Affected
- ~154 cells modified; 26 region nulls filled; 22 ship_mode nulls retained

---

## Task 3 — Clean & Validate Dates

**Scope:** `order_date`, `ship_date`

### Issues Found
- Both columns contained a simultaneous mix of 4 date formats:
  - `MM/DD/YYYY` — 247 order / 216 ship
  - `DD-MM-YYYY` — 232 / 253
  - `DD Mon YYYY` — 227 / 236
  - `YYYY-MM-DD` — 226 / 227
- 21 rows where `ship_date` was earlier than `order_date` (chronological impossibility)
- 0 missing or unrecognised dates

### Cleaning Actions
1. Identified each row's format using strict regex pattern matching (no ambiguous guessing)
2. Parsed all dates to standardised `YYYY-MM-DD` format
3. Created `order_date_std` and `ship_date_std` beside originals (originals preserved)
4. Flagged 21 rows with `date_flag = SHIP BEFORE ORDER`; no correction applied
5. Computed `shipping_delay_days = ship_date_std − order_date_std`; set NULL for the 21 flagged rows

### New Columns Added
`order_date_std`, `ship_date_std`, `date_flag`, `shipping_delay_days`

### Assumptions Made
- `DD-MM-YYYY` assumed (not `MM-DD-YYYY`) for hyphen-separated dates — validated by day values > 12 in the first segment.
- No dates were corrected or inferred; anomalous dates flagged for manual review only.

### Records Flagged
- 21 rows: `date_flag = SHIP BEFORE ORDER` (day differences −1 to −4, consistent with a date field swap)

---

## Task 4 — Handle Duplicates

**Scope:** Full 932-row dataset

### Issues Found
- 40 exact duplicate rows (20 pairs where all 25 data columns were byte-for-byte identical)
- 24 conflicting duplicate rows (12 pairs: same `order_id`, different `sales` and/or `order_status`)

### Cleaning Actions
1. **Exact duplicates:** First occurrence retained; 20 second occurrences permanently removed. Row count: 932 → 912.
2. **Conflicting duplicates:** No rows deleted. All 24 flagged with `duplicate_flag = CONFLICT - MANUAL REVIEW`.
3. Removed rows fully documented in `data_quality_report.xlsx → Records_Removed` (original row numbers preserved).

### Pattern Observed
All 12 conflicting pairs differ in exactly two columns: `sales` (minor difference) and `order_status` (Completed/Cancelled vs Returned). Consistent with return transactions written back to the original `order_id` rather than creating a new record.

### Records Removed
- **20 rows** (exact duplicates)

### Records Flagged
- 24 rows (`duplicate_flag = CONFLICT - MANUAL REVIEW`)

### Output Files
`cleaned_orders_task4.xlsx`, `data_quality_report.xlsx` (5-sheet QA workbook)

---

## Task 5 — Apply Business Rules

**Scope:** `region`, `ship_mode`, `discount`, `order_status`, `payment_status`, `date_flag`

### Business Rules Applied

| Rule | Description | Action | Records Affected |
|---|---|---|---|
| BR1 | Missing region → "Unknown" | Verified; 0 nulls remained after Task 2 | 0 |
| BR2 | Missing ship_mode → "Unknown" | `ship_mode_clean` filled with "Unknown" | 21 rows |
| BR3 | Null discount → 0 if qty/price/sales all present | `discount_clean = 0.0` | 4 rows |
| BR4 | Discount outside [0, 1] = invalid | Flagged only; not corrected | 23 rows |
| BR5 | qty × price × (1−disc) ≠ sales → set discount to 0, flag | `discount_clean = 0`; `sales_valid_flag = MISMATCH` | 41 rows |
| BR6 | Cancelled orders excluded from completed sales | `contributes_to_sales = NO - Cancelled` | 145 rows |
| BR7 | Failed payment orders excluded from completed sales | `contributes_to_sales = NO - Failed Payment` | 37 rows (net unique) |
| BR8 | Returned orders summarised separately | `contributes_to_sales = REFUND_SUMMARY` | 128 rows |
| BR9 | Ship date before order date → invalid shipping record | `shipping_record_flag = INVALID` | 21 rows |

### Issues Found
- 15 rows: negative discount (−0.24 to −0.02) — possible sign errors or surcharges
- 8 rows: discount stored as text (`"70%"`, `"85%"`) — invalid format
- 41 rows: valid-range discount but formula check fails beyond ±$0.02 tolerance
- 23 rows with invalid discount: `calculated_sales` left NULL (no fabrication)

### New Columns Added
`region_clean`, `ship_mode_clean`, `discount_clean`, `discount_flag`, `sales_valid_flag`, `order_exclusion_flag`, `contributes_to_sales`, `shipping_record_flag`

### Assumptions Made
- Tolerance for sales formula check set at ±$0.02 for floating-point rounding
- Where a row is both Returned and Failed Payment, payment failure takes exclusion priority in `contributes_to_sales`
- No numerical values were imputed or corrected

---

## Task 6 — Create Calculated Columns

**Scope:** New derived columns (non-destructive; inserted beside source columns)

### Columns Created

| Column | Formula / Logic | Nulls |
|---|---|---|
| `cleaned_discount` | Validated discount; NULL where invalid (BR4); 0 where BR3/BR5 applied | 23 |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount)` | 23 |
| `calculated_profit` | `calculated_sales − cost` | 23 |
| `profit_margin` | `calculated_profit / calculated_sales` | 23 |
| `shipping_delay_days` | `ship_date_std − order_date_std` in days | 21 |
| `order_month` | Month extracted from `order_date_std` | 0 |
| `order_year` | Year extracted from `order_date_std` | 0 |
| `data_quality_flag` | Clean / Warning / Invalid based on all prior validation flags | 0 |

### Data Quality Flag Breakdown
- **Clean:** 814 rows (89.3%) — no validation issues
- **Invalid:** 66 rows (7.2%) — invalid discount, unverifiable sales, or invalid ship date
- **Warning:** 32 rows (3.5%) — sales mismatch, duplicate conflict, excluded from sales, or unknown ship mode

### Issues Found
- 23 rows where `calculated_sales` could not be computed (invalid `cleaned_discount`)
- `profit_margin` ranges from 1.9% to 58.7%; no negative margins in completed orders

### Assumptions Made
- `unit_price` was present in the source; `unit_price = sales / quantity` formula was not needed
- `shipping_delay_days` left NULL (not negative) for SHIP BEFORE ORDER rows to prevent downstream misuse

---

## Task 7 — Final Validation

### Actions Performed
- Confirmed `profit_margin` consistent with `calculated_profit / calculated_sales` across all non-null rows
- Verified `order_month` and `order_year` correctly extracted
- Confirmed no new issues introduced by Task 6 additions
- Final row count verified: 912 rows (unchanged since Task 4)

---

## Task 8 — Pivot Summary Report

### Pivot Tables Created (pivot_summary.xlsx)

| Pivot | Filter | Sort | Key Finding |
|---|---|---|---|
| Sales & Profit by Region | `contributes_to_sales = YES` | ↓ Revenue | South leads revenue; East leads margin (30.3%) |
| Sales & Profit by Category | `contributes_to_sales = YES` | ↓ Sales within category | Technology leads; Machines sub-category highest margin (34.6%) |
| Order Count by Ship Mode | All 912 rows | ↓ Order count | Modes near-even (22–27%); 21 Unknown unresolved |
| Profit Margin by Segment | `contributes_to_sales = YES` | ↓ Margin % | Home Office 30.2%; Small Business 28.0% |
| Exception Orders by Region | Cancelled + Failed + Returned | ↓ Total exceptions | North has most exceptions (100); 310 total = 34.0% |
| Monthly Sales Trend | `contributes_to_sales = YES` | Chronological | Jun 2024 & Feb 2025 are peak months |

---

## Summary: Records Removed

| Task | Count | Reason |
|---|---|---|
| Task 4 — Exact Duplicates | 20 rows | All 25 columns identical to a prior occurrence |
| All other tasks | 0 rows | Issues flagged only; no permanent deletions |

---

## Summary: Records Flagged (912-row final dataset)

| Flag Value | Column | Count |
|---|---|---|
| CONFLICT - MANUAL REVIEW | `duplicate_flag` | 24 |
| SHIP BEFORE ORDER | `date_flag` | 21 |
| INVALID / INVALID_FORMAT | `discount_flag` | 23 |
| MISMATCH→SET_TO_0 | `discount_flag` | 41 |
| MISMATCH / UNVERIFIABLE | `sales_valid_flag` | 64 |
| INVALID (shipping) | `shipping_record_flag` | 21 |
| NO - Cancelled | `contributes_to_sales` | 145 |
| NO - Failed Payment | `contributes_to_sales` | 37 |
| REFUND_SUMMARY | `contributes_to_sales` | 128 |
| Unknown | `ship_mode_clean` | 21 |
| Invalid | `data_quality_flag` | 66 |
| Warning | `data_quality_flag` | 32 |

---

## Limitations of the Cleaning Process

1. **Source system inaccessible:** 21 `ship_mode` nulls, 21 chronologically invalid shipping dates, and 24 conflicting duplicate rows cannot be resolved without the originating transaction system.
2. **Discount invalidity unresolved:** 23 rows with invalid discounts have NULL `calculated_sales` and are excluded from all revenue and margin aggregations.
3. **No external reference data:** Region assignment for null rows relied solely on existing dataset patterns. Edge cases involving cities that genuinely span multiple regions may be incorrect.
4. **Partial October 2025 data:** Only 10 completed orders in October 2025 — the export was likely taken mid-period. Year-to-date 2025 figures should not be compared directly against full-year 2024 figures.
5. **Conflict duplicates unresolved:** 24 rows remain in the dataset but may represent either data duplicates or legitimate return events. Resolution requires manual review against source records.
6. **No PII masking:** `customer_name` and `customer_id` were cleaned for consistency but not anonymised.
