---
name: step1-clean-data
description: Step 1 of the monthly sales pipeline. Validates the month's raw sales and inventory-cost CSVs, records a raw baseline, fixes format inconsistencies into new *_clean.csv files in the output folder, and reports what changed. Use whenever new input data arrives.
---

# step1-clean-data

You clean the monthly CSV files the user drops in the `input` folder. Follow these steps every time. Do not ask questions first; do the work, then report.

## Folders
- Working folder (on the user's computer): `<PROJECT_FOLDER>`
- Inputs: `input\` (never edit, rename, move or overwrite these)
- Outputs: `output\` (cleaned files go here; create it if missing; never overwrite another month's files)

## Month and year
Follow the "Month and year detection" rules in `CLAUDE.md`. Derive `<month>` (lowercase name, for example `may`), `<year>` (four digits) and the expected month number from the input filenames, never from the system date. Use the same `<month>_<year>` in every output filename. Only touch files for that month.

## Expected files
Names follow `sales_<month>_<year>.csv` and `inventory_costs_<month>_<year>.csv` (for example `sales_may_2026.csv`). Write outputs as `sales_<month>_<year>_clean.csv` and `inventory_costs_<month>_<year>_clean.csv`.

Expected columns (if the columns differ, show the new columns to the user and adapt the rules below):
- Inventory: `SKU, Product Name, Category, Unit Cost`
- Sales: `Order ID, Customer Name, Product SKU, Order Date, Revenue`

## Steps
1. List the `input\` folder with `device_list_dir`, work out the month and year (see Month and year), then copy only that month's two CSVs into the workspace with `device_stage_files`. Record each input file's size and modified time (and a SHA-256 hash of the staged copy).
2. **Validate the input files before anything else.** Stop and report, producing no outputs for the month, if a file: is missing; is not plain CSV text (for example it contains `<html`, an error page, or binary content); is empty; or does not have exactly the expected header columns. Say which file failed and why. Do not try to repair or guess.
3. Read every row of every file. Do not rely on `head`. Use `cat -A` to catch stray spaces and line-ending problems.
4. **Record the raw baseline** from the untouched files (read as strings, convert the numeric columns only for the totals): number of sales rows, sum of Revenue, number of inventory rows, and each SKU's Unit Cost, Product Name and Category exactly as given. This baseline is the source of truth for the reconciliation in steps 2 and 3.
5. Apply the fixes below with a pandas script. Read all columns as strings (`dtype=str`) so nothing is silently converted.
6. Run the checks below and record the counts of changed rows per column.
7. Write the clean files to the workspace output folder, then copy them to the `output\` folder with `device_commit_files`.
8. Confirm the input files are unchanged (same size, modified time and hash as in step 1).
9. Report to the user (see Report).

## Fixes (the standard formats)
| Column | Standard | Examples of what to fix |
|---|---|---|
| SKU / Product SKU | `PREFIX-NNN`, uppercase, hyphen between letters and digits | `SHOE001` -> `SHOE-001`, `shoe-001` -> `SHOE-001` |
| Order ID | `ORD-NNNN` | `ORD1002` -> `ORD-1002` |
| Customer Name | Title Case, trimmed | `jane smith` -> `Jane Smith` |
| Order Date | ISO `YYYY-MM-DD` | `01/05/26` -> `2026-05-01` |

Date rule: the source dates have been `dd/mm/yy` (day first). Confirm this by checking that the day goes above 12 and that all dates fall in one month. If the format looks different in a new file, say so in the report instead of guessing.

Fix SKUs in both files the same way so that sales join to inventory.

## Checks (run after fixing)
- No duplicate Order IDs and no duplicate SKUs.
- Every Product SKU in sales exists in inventory. List any that do not.
- All dates fall inside the month and year taken from the filename. List any that do not.
- **Cleaning changed no numbers:** clean sales rows equal raw sales rows, the sum of clean Revenue equals the raw sum, and every SKU's Unit Cost, Product Name and Category in the clean inventory equals the raw value (only the SKU text format is changed).
- The input files are unchanged.
- No order has revenue below the unit cost. List any that do.
- No blank values or non-numeric Revenue / Unit Cost.
- Inventory SKUs with no sales (just report; this can be normal).

## Flag, do not change
Report these to the user and leave the data as it is:
- A product whose category looks wrong (for example `Sports Socks` under Footwear). Never change a category, product name or cost.
- Customer names that look abbreviated (`Priya M`) or that may be the same person under different spellings.
- Any missing quantity column that stops revenue being compared with unit cost per unit.
- Anything else that looks wrong but where the right answer is a business decision.

## Report
Reply in plain language with:
1. The month and year detected and the files used.
2. The raw baseline: sales rows, total revenue, inventory rows.
3. What was fixed, per file and per column, with the number of rows changed.
4. Which checks passed and which found problems.
5. The flagged items the user needs to decide.
6. Where the clean files are (the `output\` folder).

Keep it short. Do not recap the steps you took.

## History
- 2026-11-06: First run on May 2026 data. Fixes: 6 inventory SKUs, 68 sales SKUs, 60 Order IDs, 27 customer names, 120 dates. All checks passed. Flagged: Sports Socks category, abbreviated names, 5 unsold SKUs, no quantity column.
- 2026-11-09: Instructions updated: added month and year detection, input validation (stop on a non-CSV or wrong-header file), raw baseline for reconciliation, and an input-unchanged check. No May outputs were regenerated.
- 2026-11-09: Run on June 2026 data (first June inventory file was a 504 error page and was rejected; corrected file validated). Fixes: 6 inventory SKUs, 66 sales SKUs, 60 Order IDs, 29 customer names, 120 dates. All checks passed. Flagged: Sports Socks category, abbreviated names, 5 unsold SKUs, order ORD-2022 revenue (1,842) below unit cost (1,900), no quantity column.
