---
name: step4-1-dashboard
description: Builds a single self-contained HTML sales dashboard (KPIs, category, product, customer, weekly charts and a sortable product table) from the summary and merged CSVs. Use after step3-summary has produced the summary CSV.
---

# step4-1-dashboard

You turn the summary CSV and the merged CSV into one self-contained HTML dashboard. Run this after `step3-summary.md`. Do not ask questions first; do the work, then report.

## Folders
- Working folder (on the user's computer): `<PROJECT_FOLDER>`
- Inputs: `output\summary_<month>_<year>.csv` and `output\merged_sales_inventory_<month>_<year>.csv`
- Output: `output\dashboard_<month>_<year>.html`

If either input is missing, run the earlier step files (`step1-clean-data.md`, `step2-merge-data.md`, `step3-summary.md`) first.

## Month and year and start condition
Follow the "Month and year detection" rules in `CLAUDE.md` and derive `<month>` and `<year>` from the summary filename. Start only after `step3-summary.md` has finished and written both `summary_<month>_<year>.csv` and `summary_<month>_<year>_formatted.xlsx`. This step runs in parallel with `step4-2-executive-ppt.md` as a separate agent; it only reads from `output\` and writes only its own file, so the two never touch the same file. Never overwrite another month's output.


## Expected input
- Summary CSV (long format) columns: `Level, Name, Orders, Revenue, Avg_Order_Value, Revenue_Share_Pct, Est_Cost, Est_Margin, Est_Margin_Pct, Rank_In_Level`, with levels `Overall, Category, Product, Customer, Week`.
- Merged CSV columns: `Order ID, Customer Name, Order Date, Product SKU, Product Name, Category, Unit Cost, Revenue`.
If the columns differ, show them to the user and adapt.

## Steps
1. List `output\` with `device_list_dir` and copy both CSVs into the workspace with `device_stage_files`.
2. Load the `dataviz` skill before writing any chart code and follow it (single hue for magnitude bars, no dual axis, direct value labels, hover tooltips, light mode, Arial).
3. Verify the inputs, do not trust them. With pandas, recompute from the merged file: order count, total revenue, total cost, average order value, and per Category, Product and Customer the orders, revenue and cost. Stop and report if any differs from the summary CSV, or if the Week revenues do not add up to total revenue.
4. Embed the summary rows as JSON inside the HTML (one list per level). The page must have no external network dependencies: inline CSS, JS and SVG only.
5. Build the page with these parts, in this order: KPI row (orders, revenue, est. cost, est. gross profit, est. margin %, average order value); revenue by category (with share %); weekly revenue line; revenue by product; estimated margin % by product (a separate chart, never a second axis); top customers by revenue (all customers, highest first); sortable product table (click a header to sort).
6. Add a visible footnote: cost, gross profit and margin are estimates because the sales data has no quantity column, so cost assumes one unit per order.
7. Write the file to `<WORKSPACE_OUTPUT>/dashboard_<month>_<year>.html`, then copy it to `output\` with `device_commit_files`.
8. Render check: parse the script (`node --check`), open it in headless Chrome, take a screenshot and look for overlapping labels or clipped text, and confirm the KPI values in the rendered page match the CSV.
9. Report to the user.

## Checks
- Orders, revenue, est. cost, est. gross profit, margin % and average order value on the dashboard equal the CSV values, the merged-file recomputation and the figures reconciled to the raw source data in step 3.
- Every chart value and table cell matches the summary row it came from.
- The footnote about estimated cost is visible.
- The file opens with no network access and the script has no errors.
- If the month has days 29-31, the summary has a Week 5 row; make sure the weekly chart shows it.

## Report
Reply in 3 or 4 lines: the file name written, the headline figures shown (orders, revenue, gross profit, margin %, average order value), the caveat that cost and margin are estimates, and anything that looked wrong. Offer one next step at most.

## History
- 2026-11-09: First run on May 2026 data. 120 orders, revenue 521,616, est. cost 161,600, est. gross profit 360,016, margin 69.02%, average order value 4,346.80. All figures matched the merged-file recomputation. Rendered and checked in headless Chrome.
- 2026-11-09: Instructions updated: month detection, start condition (after step 3) and parallel run with step 4-2, and a check against the step 3 reconciliation. No May outputs were regenerated.
- 2026-11-09: Run on June 2026 data. 120 orders, revenue 505,449, est. cost 162,800, est. gross profit 342,649, margin 67.79%, AOV 4,212.08. Matched recomputation; rendered and checked in headless Chrome.
