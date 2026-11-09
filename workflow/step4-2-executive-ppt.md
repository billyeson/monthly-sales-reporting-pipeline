---
name: step4-2-executive-ppt
description: Builds a one-slide, 16:9 executive PowerPoint (headline, KPI strip, two native charts, insight bullets, estimate footnote) from the summary and merged CSVs. Use after step3-summary has produced the summary CSV.
---

# step4-2-executive-ppt

You turn the summary CSV into one executive slide. Run this after `step3-summary.md`. Do not ask questions first; do the work, then report. Load the Skill `anthropic-skills:pptx` before building and follow it.

## Folders
- Working folder (on the user's computer): `<PROJECT_FOLDER>`
- Inputs: `output\summary_<month>_<year>.csv` and `output\merged_sales_inventory_<month>_<year>.csv`
- Output: `output\executive_summary_<month>_<year>.pptx`

If either input is missing, run `step1-clean-data.md`, `step2-merge-data.md` and `step3-summary.md` first.

## Month and year and start condition
Follow the "Month and year detection" rules in `CLAUDE.md` and derive `<month>` and `<year>` from the summary filename. Start only after `step3-summary.md` has finished and written both `summary_<month>_<year>.csv` and `summary_<month>_<year>_formatted.xlsx`. This step runs in parallel with `step4-1-dashboard.md` as a separate agent; it only reads from `output\` and writes only its own file, so the two never touch the same file. Never overwrite another month's output.


## Expected input
- Summary CSV columns: `Level, Name, Orders, Revenue, Avg_Order_Value, Revenue_Share_Pct, Est_Cost, Est_Margin, Est_Margin_Pct, Rank_In_Level`, with levels `Overall, Category, Product, Customer, Week`.
- Merged CSV columns: `Order ID, Customer Name, Order Date, Product SKU, Product Name, Category, Unit Cost, Revenue`.
If the columns differ, show them to the user and adapt.

## Steps
1. List `output\` with `device_list_dir` and copy both CSVs into the workspace with `device_stage_files`.
2. Verify the numbers before building. Recompute from the merged CSV with pandas: order count, total revenue, total Unit Cost, gross profit (revenue - cost), margin %, average order value, revenue share by category, top product, top customer, and revenue per week (days 1-7, 8-14, 15-21, 22-end). Compare with the `Overall`, `Category`, `Product`, `Customer` and `Week` rows of the summary. Stop and report if anything differs. Do not trust claimed figures; use the CSV values.
3. Pick the content from the summary CSV (read it in code; never retype numbers):
   - Headline: states the takeaway, for example `<top category> drove <share>% of <month> revenue; <margin>% estimated margin on <revenue> revenue`.
   - KPI strip (5 cards): Orders, Revenue, Est. cost*, Est. gross profit*, Est. margin* from the `Overall` row.
   - Chart 1: horizontal bar, revenue by category (`Category` rows, highest first).
   - Chart 2: column chart, weekly revenue (`Week` rows in date order), strongest week in the accent colour.
   - 3 bullets: top product (revenue and share), top customer (revenue and share), strongest week (revenue and share).
   - Footnote: cost, gross profit and margin are estimates because the sales data has no quantity column, so cost assumes one unit per order; name the source file and order count.
   - If a future file has a quantity column, drop the `Est_` wording and the footnote's assumption.
4. Build with a `pptxgenjs` script (a short build script; `require('pptxgenjs')`, no install): `LAYOUT_WIDE` (13.33 x 7.5 in, 16:9), exactly one slide, white background, Arial everywhere, navy text `14213D`, teal `0B7A75` bars, orange `D9480F` for the highlighted week. Layout: headline 28 pt bold at top (y 0.3, h 1.0); KPI cards at y 1.5, h 1.15 (value 28 pt bold, label 13 pt); two charts at y 2.95, h 3.45, 4.2 in wide each; insights column on the right (heading 16 pt, bullets 14 pt); footnote 11 pt at y 6.65. Margins 0.5 in. Native charts only (`addChart`), data labels with `#,##0`, no gridlines, hidden value axis, no legend. Every `addText` uses `isTextBox: true`. No hex `#`.
5. Write `<WORKSPACE_OUTPUT>/executive_summary_<month>_<year>.pptx`. Run `scripts/office/validate.py` on it. If it warns about folder entries in the ZIP, remove them with `zip -dq file.pptx '*/'`.
6. Render: `scripts/office/soffice.py --headless --convert-to pdf`, then `pdftoppm -jpeg -r 80`, and look at the image. Fix any overlap, cut-off text or wrapped numbers, then re-render.
7. Copy the pptx to `output\` with `device_commit_files`.
8. Report to the user.

## Checks
Stop and report if any fail:
- Exactly 1 slide, 16:9.
- Every number on the slide matches the summary CSV, the recomputation from the merged CSV and the figures reconciled to the raw source data in step 3 (orders, revenue, cost, gross profit, margin, category chart values, weekly chart values, bullets).
- Orders in summary equal rows in merged CSV; revenue sums match.
- Footnote about estimated cost is present.
- Validator passes; rendered image shows nothing overlapping, cut off or below 0.5 in margins; font is Arial.

## Report
Reply in 3 or 4 lines: the file name written, the headline on the slide, the key numbers, and anything that looked wrong (a week or category far from the rest, a claim that did not verify). Remind the user margin is estimated. Do not recap the steps.

## History
- 2026-11-09: First run on May 2026 data. All figures verified (120 orders, revenue 521,616, est. cost 161,600, est. gross profit 360,016, margin 69.02%). Headline: Footwear drove 66.5% of May revenue; 69.02% estimated margin on 521,616 revenue. Layout rendered clean, validator passed.
- 2026-11-09: Instructions updated: month detection, start condition (after step 3) and parallel run with step 4-1, and a check against the step 3 reconciliation. No May outputs were regenerated.
- 2026-11-09: Run on June 2026 data. All figures verified. Headline: Footwear drove 67.64% of June revenue; 67.79% estimated margin on 505,449 revenue. One slide, validator passed, render checked.
