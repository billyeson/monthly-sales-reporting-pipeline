---
name: step3-summary
description: Step 3 of the monthly sales pipeline. Builds the summary CSV and the formatted Excel workbook from the merged sales + inventory file (output of step2-merge-data) and reconciles them to the raw source data. Use after step2-merge-data has produced the merged CSV.
---

# step3-summary

You turn the merged CSV into two files that later steps use: a summary CSV (`summary_<month>_<year>.csv`) and a formatted Excel workbook (`summary_<month>_<year>_formatted.xlsx`). Run this after `step2-merge-data.md`. Steps 4-1 and 4-2 start only after this step has finished and both files are written. Do not ask questions first; do the work, then report.

## Folders
- Working folder (on the user's computer): `<PROJECT_FOLDER>`
- Input: `output\merged_sales_inventory_<month>_<year>.csv` (produced by step2-merge-data)
- Raw sources for reconciliation (read only): `input\sales_<month>_<year>.csv` and `input\inventory_costs_<month>_<year>.csv`
- Outputs: `output\summary_<month>_<year>.csv` and `output\summary_<month>_<year>_formatted.xlsx`

## Month and year
Follow the "Month and year detection" rules in `CLAUDE.md`. Derive `<month>` (lowercase name, for example `may`), `<year>` (four digits) and the expected month number from the input filenames, never from the system date. Use the same `<month>_<year>` in every output filename. Only touch files for that month.

If the merged file doesn't exist, run `step1-clean-data.md` and `step2-merge-data.md` first.

## Expected input columns
`Order ID, Customer Name, Order Date, Product SKU, Product Name, Category, Unit Cost, Revenue`. If the columns differ, show them to the user and adapt.

## Steps
1. List `output\` with `device_list_dir` and copy the merged CSV into the workspace with `device_stage_files`. Also stage the month's two raw files from `input\` (read only).
2. Compute the **source figures** directly from the raw files, independently of the merged file: orders, total revenue, total cost (sum over every raw sales row of the raw Unit Cost of its SKU; one unit per order), gross profit and margin. These are the source of truth.
3. Read the merged CSV with pandas. Add a `Week` label from the order date: days 1-7 = Week 1, 8-14 = Week 2, 15-21 = Week 3, 22-28 = Week 4, 29 to month end = Week 5 (label like `Week 1 (May 1-7)`; the last label ends at the month's last day, for example `Week 5 (Jun 29-30)`). Include a Week 5 row only if there are orders on those days; never merge them into Week 4.
4. Build the summary CSV: one row per group at each level below, plus one `Overall` row. Compute the columns below.
5. Summary checks (stop and report if any fail): for every level, the sum of `Orders` equals the number of merged rows and the sum of `Revenue` equals the total revenue.
6. Build the formatted Excel workbook (see Excel workbook). Recalculate it and confirm there are no formula errors.
7. **Reconcile** (see Reconciliation). Stop and report if anything differs.
8. Write both files to the workspace output folder, then copy them to `output\` with `device_commit_files`.
9. Report to the user.

## Summary file layout
One long file with these columns:

| Column | Meaning |
|---|---|
| Level | `Overall`, `Category`, `Product`, `Customer` or `Week` |
| Name | The group (for example `Footwear`, `Running Shoes`, `Jane Smith`, `Week 1 (May 1-7)`) |
| Orders | Count of orders |
| Revenue | Sum of Revenue |
| Avg_Order_Value | Revenue / Orders, 2 decimals |
| Revenue_Share_Pct | Share of total revenue, 2 decimals |
| Est_Cost | Sum of Unit Cost over the orders (assumes one unit per order) |
| Est_Margin | Revenue - Est_Cost |
| Est_Margin_Pct | Est_Margin / Revenue, 2 decimals |
| Rank_In_Level | Rank by Revenue within the level (1 = highest) |

Row order: Overall, then Category, Product and Customer sorted by Revenue (highest first), then Week in date order.

## Excel workbook
Load the `anthropic-skills:xlsx` skill before building and follow it (openpyxl, Arial, formulas not hardcoded results, recalculate with its `recalc.py`, zero formula errors).

**Sheet `Sales Summary`** (the report), five sections stacked vertically, all starting at cell A1 with two blank rows between sections. Section titles are bold and start with `##` exactly as shown; bold column headers sit directly under each title. Revenue, cost, profit and average order value use `#,##0.00`; margins are stored as fractions and shown with `0.00%`. Column widths auto-fit the data (do not size them from the section titles, which can overflow into empty cells). No overall totals row is added to the product, category, customer or week sections.
1. `## OVERALL`: Total Orders, Total Revenue, Total Cost, Total Gross Profit, Overall Margin % (one row of values under the headers)
2. `## BY PRODUCT`: SKU, Product Name, Category, Unit Cost, Orders, Total Revenue, Total Cost, Gross Profit, Margin % (sorted by revenue, highest first; products with no sales do not appear)
3. `## BY CATEGORY`: Category, Orders, Total Revenue, Total Cost, Gross Profit, Margin % (sorted by revenue)
4. `## BY CUSTOMER`: Customer, Orders, Total Revenue, Average Order Value (sorted by revenue)
5. `## BY WEEK`: Week, Orders, Total Revenue, Total Cost, Gross Profit, Margin % (date order, including Week 5 when present)

SKU, product name, category and unit cost come from the merged file (that is, the inventory file); never invent or change them. A one-line italic note below the last section says Total Cost assumes one unit per order because the sales data has no quantity column.

**Sheet `Source Data`**: the merged rows (`Order ID, Customer Name, Order Date` as a real date `yyyy-mm-dd, Product SKU, Product Name, Category, Unit Cost, Revenue`) plus a `Week` column `=INT((DAY(C2)-1)/7)+1`. Every number on `Sales Summary` is a formula on this sheet (`COUNTA`, `SUM`, `COUNTIFS`, `SUMIFS`, `INDEX/MATCH`, `IFERROR` for margins). The overall figures are calculated from the order rows, never by adding up the product, category, customer or week sections, which overlap.

## Reconciliation
Compare the source figures (step 2 of Steps, from the raw input files) with the `Overall` row of the summary CSV and with the Overall section of the recalculated workbook (read with `data_only=True`):

| Figure | Source (raw files) | Summary CSV | Workbook | Difference |
|---|---|---|---|---|
| Orders | | | | must be 0 |
| Revenue | | | | must be 0.00 |
| Cost | | | | must be 0.00 |
| Gross profit | | | | must be 0.00 |
| Margin % | | | | must be 0.00 |

Also confirm that each section's Orders and Revenue (and Total Cost where shown) add back to the overall figures, that the summary CSV and the workbook agree on every Category, Product, Customer and Week row, and that no source category, product name or cost was changed.

## Important caveat
The sales file has no quantity column, so `Est_Cost`, `Est_Margin` and `Est_Margin_Pct` (and the workbook's cost, profit and margin) assume one unit per order. Treat them as estimates. If a future file has a quantity column, use `Unit Cost x Quantity` and drop the `Est_` prefix. Always tell the user about this caveat in the report.

## Report
Reply in plain language with:
1. The month and year detected, and the headline numbers: total orders, total revenue, average order value.
2. The reconciliation table, with the differences.
3. The top category, product and customer by revenue, and which week was highest.
4. The caveat about estimated margin.
5. Anything that looks odd (a week or customer far from the rest, products with no sales that are missing from the summary).
6. Where the two files are saved.

Keep it short. Do not recap the steps. Offer one next step at most.

## History
- 2026-11-08: First run on May 2026 data. 120 orders, revenue 521,616, average order value 4,346.80. Top category Footwear (66.5%), top product Running Shoes, top customer Jane Smith, highest week Week 1. All sum checks passed.
- 2026-11-09: The formatted Excel workbook for May 2026 (`summary_may_2026_formatted.xlsx`) was built; overall figures reconciled to the source (120 orders, revenue 521,616.00, cost 161,600.00, gross profit 360,016.00, margin 69.02%). Instructions updated: the workbook, reconciliation to raw files, Week 5 handling and month detection are now part of this step. No May outputs were regenerated.
- 2026-11-09: Run on June 2026 data. 120 orders, revenue 505,449, average order value 4,212.08. Top category Footwear (67.64%), top product Casual Shoes, top customer Ravi Mehta, highest week Week 4 (no Week 5). CSV and workbook reconciled to raw with zero differences; workbook recalculated with 0 formula errors.
