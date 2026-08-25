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
delimited)" and re-imported into MySQL. [Update this line once you've 
confirmed the new row count — see note below.]

**Why this matters:**
This kind of silent row loss during import doesn't throw an error — it just 
quietly gives you fewer rows than you expect. If unnoticed, it could skew 
every downstream total (sales, profit, etc.) without any obvious sign 
something was wrong. Always verify row counts immediately after importing 
data, before doing any analysis.
