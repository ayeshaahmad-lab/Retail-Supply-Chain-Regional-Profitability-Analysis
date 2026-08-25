# Retail-Supply-Chain-Regional-Profitability-Analysis
End-to-end data cleaning and exploratory data analysis of a retail supply chain dataset using MySQL, featuring advanced SQL window functions, CTEs, and business insights.
## Data Cleaning

### Issue 1: Row count mismatch after import

**What I found:**
The raw CSV file contained 9,994 data rows (9,995 lines including the header). 
After importing into MySQL via Workbench's Table Data Import Wizard, the 
`sales` table only contained 9,694 rows — 300 rows short.

**Investigation:**
I ran the following query to check whether the missing rows were duplicates, 
a clean truncation at the end, or scattered gaps:

```sql
SELECT 
    MIN(`Row ID`) AS min_id, 
    MAX(`Row ID`) AS max_id, 
    COUNT(DISTINCT `Row ID`) AS distinct_ids,
    COUNT(*) AS total_rows
FROM sales;
```

Result: `min_id = 1`, `max_id = 9994`, `distinct_ids = 9694`, `total_rows = 9694`.

Since `distinct_ids` equals `total_rows`, there were no duplicate rows. But 
since `max_id` (9994) was much higher than `distinct_ids` (9694), the missing 
rows were not a simple end-truncation — they were gaps scattered throughout 
the dataset. I confirmed this by generating a full sequence of numbers from 
1–9994 and comparing it against the existing Row IDs to find exactly which 
ones were missing, and found the missing IDs were randomly distributed 
rather than clustered.

**Likely cause:**
The source file had been converted from an Excel worksheet to CSV before 
import. This is a known source of silent row loss because:
- Excel's default CSV export can use Windows-1252 encoding instead of UTF-8, 
  which can break on special characters (e.g. accented names).
- Fields containing commas (e.g. product names like "Hon Deluxe Fabric 
  Upholstered Stacking Chairs, Rounded Back") can be improperly escaped, 
  causing the importer to misread row boundaries.

**Fix:**
Re-exported the source file from Excel specifically as "CSV UTF-8 (Comma 
delimited)" and re-imported into MySQL.
After multiple failed import attempts via MySQL Workbench's Table Data Import 
Wizard (which silently dropped rows in different ways — first 300 missing 
rows scattered throughout the file, then as few as 160 rows imported after 
re-exporting as CSV UTF-8), I abandoned the import wizard entirely. Instead, 
I generated a SQL script containing explicit `CREATE TABLE` and `INSERT` 
statements built directly from the validated CSV data, and ran it as a 
script in Workbench. This guaranteed all 9,994 rows loaded correctly, since 
the data no longer depended on any tool's CSV parsing behavior.

Verified via:
```sql
SELECT COUNT(*) FROM sales;
```
Result: 9994 — matching the raw file's row count exactly.

**Why this matters:**
This kind of silent row loss during import doesn't throw an error — it just 
quietly gives you fewer rows than you expect. If unnoticed, it could skew 
every downstream total (sales, profit, etc.) without any obvious sign 
something was wrong. Always verify row counts immediately after importing 
data, before doing any analysis.

### Issue 2: Duplicate rows

**What I checked:**
1. Whether `Row ID` (the assumed primary key) contained any duplicate values.
2. Whether any full rows were duplicated — i.e. the same order/customer/
   product/sales data repeated under a different `Row ID`.

**Query 1 — Row ID duplicates:**
```sql
SELECT `Row ID`, COUNT(*)
FROM sales
GROUP BY `Row ID`
HAVING COUNT(*) > 1;
```
Result: no duplicate `Row ID` values found — confirming the primary key is clean.

**Query 2 — Full row duplicates:**
Since a duplicate could still exist under a different `Row ID`, I used a window 
function to check for rows where every column (excluding `Row ID`) matched 
another row exactly:

```sql
SELECT *, ROW_NUMBER() OVER (
    PARTITION BY `Order ID`, `Order Date`, `Ship Date`, `Ship Mode`, 
    `Customer ID`, `Customer Name`, `Segment`, `Country`, `City`, `State`, 
    `Postal Code`, `Region`, `Retail Sales People`, `Product ID`, `Category`, 
    `Sub-Category`, `Product Name`, `Returned`, `Sales`, `Quantity`, 
    `Discount`, `Profit`
) AS row_num
FROM sales;
```

Rows where `row_num > 1` indicate a true duplicate.

**Result:** Found **1 duplicate row**.

**How I handled it:**
Rather than deleting directly from the original `sales` table, I created a 
second table, `sales_clean`, containing a full copy of the data plus the 
`row_num` result above. I then removed the duplicate row from `sales_clean` 
only. This keeps the original `sales` table as an untouched, raw backup — so 
if I ever need to verify a cleaning decision or start over, the original 
import is still intact. From this point onward, all further cleaning and 
analysis in this project is done on `sales_clean`, not `sales`.

**Why this matters:** Even a single duplicate row can silently inflate sales, 
quantity, or profit totals in later aggregate analysis. Keeping a raw, 
unmodified copy of the original data alongside the cleaned version is a 
standard practice — it makes every cleaning decision traceable and reversible.
