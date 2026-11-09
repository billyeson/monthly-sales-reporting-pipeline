---
name: step2-merge-data
description: Step 2 of the monthly sales pipeline. Merges the cleaned sales and inventory-cost CSVs (output of step1-clean-data) by SKU into one file with one row per order, and reconciles it to the raw source data. Use after step1-clean-data has produced the *_clean.csv files.
---

# step2-merge-data

You merge the two cleaned monthly CSVs into one file. Run this after `step1-clean-data.md`. Do not ask questions first; do the work, then report.

## Folders
- Working folder (on the user's computer): `<PROJECT_FOLDER>`
- Inputs: the `*_clean.csv` files in `output\` (produced by step1-clean-data). Never use the raw files in `input\`.
- Raw sources for reconciliation (read only): `input\sales_<month>_<year>.csv` and `input\inventory_costs_<month>_<year>.csv`
- Output: `output\merged_sales_inventory_<month>_<year>.csv`

## Month and year
Follow the "Month and year detection" rules in `CLAUDE.md`. Derive `<month>` (lowercase name, for example `may`), `<year>` (four digits) and the expected month number from the input filenames, never from the system date. Use the same `<month>_<year>` in every output filename. Only touch files for that month.

## Expected files
- `sales_<month>_<year>_clean.csv` with `Order ID, Customer Name, Product SKU, Order Date, Revenue`
- `inventory_costs_<month>_<year>_clean.csv` with `SKU, Product Name, Category, Unit Cost`

If the clean files don't exist yet, run `step1-clean-data.md` first. If the columns differ, show them to the user and adapt.

## Steps
1. List `output\` with `device_list_dir` and copy the month's two clean CSVs into the workspace with `device_stage_files`. Also stage the month's two raw files from `input\` for the reconciliation (read only).
2. Read both with pandas as strings (`dtype=str`).
3. Left join sales to inventory: `sales['Product SKU']` = `inventory['SKU']`. Use `validate='m:1'` so a duplicate SKU in inventory fails loudly instead of duplicating orders.
4. Drop the duplicate `SKU` column and order the columns as: `Order ID, Customer Name, Order Date, Product SKU, Product Name, Category, Unit Cost, Revenue`.
5. Checks (stop and report if any fail):
   - Every sales row matched an inventory row (no unmatched SKUs).
   - Row count equals the sales row count (no rows lost or duplicated).
   - No blank values in the merged file.
   - **Reconcile to the raw source data** (see Reconciliation). Stop and report if anything differs.
6. Write the merged file to the workspace output folder, then copy it to `output\` with `device_commit_files`.
7. Report to the user.

## Reconciliation (independent of the merge code)
Recompute from the raw `input\` files, using the SKU text only as a lookup key (normalise it as in step 1), and compare to the merged file:
- Merged rows = raw sales rows.
- Sum of merged Revenue = sum of raw Revenue (exact).
- Sum of merged Unit Cost = sum, over every raw sales row, of the raw Unit Cost of that row's SKU. This is the estimated total cost (one unit per order; the sales data has no quantity column).
- Gross profit = revenue - cost, and margin = gross profit / revenue. Print these four figures; step 3 and the reports must show the same numbers.
- Every merged Product Name, Category and Unit Cost equals the raw inventory value for that SKU. Do not change a source category.

## Report
Reply in plain language with:
1. What was joined, and on which columns.
2. Row counts (raw sales, merged out), whether every order matched, and the reconciliation figures (revenue, estimated cost, gross profit, margin) with the differences, which must be zero.
3. The columns in the merged file.
4. Inventory products that have no sales (they don't appear in the merged file).
5. Where the file is saved.

Keep it short. Do not recap the steps. Offer one next step at most, such as a margin column (Revenue minus Unit Cost) or a summary by product, customer or category. Do not add extra columns unless asked.

## History
- 2026-11-08: First run on May 2026 data. 120 sales rows joined to inventory, all matched, 120 rows out, no blanks. Five products had no sales: TSHIRT-002, JACKET-001, BELT-001, CAP-001, SOCK-001.
- 2026-11-09: Instructions updated: month and year detection, and a reconciliation of orders, revenue, cost, gross profit and margin back to the raw input files. No May outputs were regenerated.
- 2026-11-09: Run on June 2026 data. 120 sales rows joined, all matched, 120 rows out, no blanks. Reconciled to raw: revenue 505,449, est. cost 162,800, gross profit 342,649, margin 67.79%, all differences 0. Unsold: TSHIRT-002, JACKET-001, BELT-001, CAP-001, SOCK-001.
