# Monthly Sales Reporting: master reference

This project turns two raw monthly CSVs (sales and inventory costs) into a cleaned dataset, a merged file, a summary, a formatted Excel workbook, an HTML dashboard and a one-slide executive PowerPoint. This file is the master reference. Each stage has its own instruction file; read the step file before running that stage and follow it.

Working folder (on the user's computer): `<PROJECT_FOLDER>`

> Repository note: in this published repo the step files live in `workflow/`. The folder layout below describes the working folder used when the pipeline runs: `input\` corresponds to `data/input/` here, and `output\` corresponds to `data/intermediate/` (clean and merged files), `reports/` (summary CSV, Excel, PowerPoint) and `docs/` (dashboards). When running the pipeline, copy the step files next to this file in your `<PROJECT_FOLDER>` (see README), or adjust the paths.

## Folder structure
```
20261106_wfc_claude_cowork\
  CLAUDE.md                  this master reference
  step1-clean-data.md        Step 1 instructions
  step2-merge-data.md        Step 2 instructions
  step3-summary.md           Step 3 instructions (summary CSV + formatted Excel)
  step4-1-dashboard.md       Step 4-1 instructions
  step4-2-executive-ppt.md   Step 4-2 instructions
  input\                     raw files from the user (never edited)
    sales_<month>_<year>.csv
    inventory_costs_<month>_<year>.csv
  output\                    everything generated
    sales_<month>_<year>_clean.csv                     Step 1
    inventory_costs_<month>_<year>_clean.csv           Step 1
    merged_sales_inventory_<month>_<year>.csv          Step 2
    summary_<month>_<year>.csv                         Step 3
    summary_<month>_<year>_formatted.xlsx              Step 3
    dashboard_<month>_<year>.html                      Step 4-1
    executive_summary_<month>_<year>.pptx              Step 4-2
```
Never create another CLAUDE.md or another copy of a step file. Update these files in place.

## The pipeline
| Step | Instruction file | Reads | Writes (in `output\`) |
|---|---|---|---|
| 1. Clean | `step1-clean-data.md` | `input\` raw sales and inventory CSVs | the two `*_clean.csv` files |
| 2. Merge by SKU | `step2-merge-data.md` | the two clean files (and the raw files to reconcile) | `merged_sales_inventory_<m>_<y>.csv` |
| 3. Summary and Excel | `step3-summary.md` | merged file (and the raw files to reconcile) | `summary_<m>_<y>.csv`, `summary_<m>_<y>_formatted.xlsx` |
| 4-1. Dashboard | `step4-1-dashboard.md` | summary + merged | `dashboard_<m>_<y>.html` |
| 4-2. Executive slide | `step4-2-executive-ppt.md` | summary + merged | `executive_summary_<m>_<y>.pptx` |

### Execution order
1. Run Steps 1, 2 and 3 one after the other, in that order. Do not start a step until the previous one has passed its checks. If a step fails, stop, report what failed and do not produce later outputs.
2. Only after Step 3 has finished and both its files are written, run Steps 4-1 and 4-2 in parallel as two agents (one per step file), launched together. They read from `output\` and each writes only its own file.
3. When both agents finish, check their results (the dashboard rendered, the slide rendered, figures match), copy the files to `output\`, then give one short summary.

## Month and year detection
- Input files are named `sales_<month>_<year>.csv` and `inventory_costs_<month>_<year>.csv` (month is a full English name in lowercase, year has four digits, for example `sales_may_2026.csv`). Match names without regard to case.
- The month key is `<month>_<year>`. A month is ready when both of its files exist. Use that key in every output filename.
- If the user's request names a month and year, use it. If it names none and exactly one ready month has no outputs yet, use that month. If several months qualify, list them and ask which one. Never process a month the user did not ask for.
- Take the expected month and year for date checks from the filename, never from the system clock.
- Only ever read the chosen month's input files and write the chosen month's outputs. Never open, modify or overwrite another month's input or output files.
- If a file is missing, misnamed or has the wrong columns, stop and tell the user what is wrong.

## Input validation
Before cleaning, each raw file must be real plain-text CSV with exactly the expected header (`Order ID, Customer Name, Product SKU, Order Date, Revenue` for sales; `SKU, Product Name, Category, Unit Cost` for inventory), at least one data row, and no web page or error text inside. A file that fails is reported and not repaired; no outputs are produced for that month until the user supplies a good file.

## Reconciliation to the source data
The raw files in `input\` are the source of truth. Recompute these from them independently of the pipeline code:
- Orders = number of raw sales rows.
- Revenue = sum of raw Revenue.
- Cost = sum, over every raw sales row, of the raw Unit Cost of that row's SKU (one unit per order).
- Gross profit = revenue - cost. Margin = gross profit / revenue.

These must match exactly (difference 0) in the merged file (Step 2), the summary CSV and the Excel workbook (Step 3), and the figures shown on the dashboard and the slide (Steps 4-1 and 4-2). Product names, categories and unit costs must equal the inventory file values. Overall totals come from the order rows, never from adding up the product, category, customer or week sections. Any difference stops the pipeline and is reported.

## Standing rules and assumptions (do not re-ask)
- **Cost is an estimate.** The sales data has no quantity column, so Total Cost = Unit Cost x number of orders (one unit per order). Label cost, profit and margin as estimates and state the assumption in every deliverable. If a future file has a quantity column, use Unit Cost x Quantity and drop the "Est." labels.
- **Original files stay unchanged.** Never edit, rename, move or overwrite anything in `input\`, and never overwrite another month's outputs. Check that the input files' size and modified time are unchanged after Step 1.
- **Standard formats:** SKU `PREFIX-NNN` uppercase; Order ID `ORD-NNNN`; customer names in Title Case; dates `YYYY-MM-DD` (source dates have been day-first `dd/mm/yy`; if the format looks different, say so instead of guessing).
- **Never invent data, and never silently change source categories, product names or costs.** SKU, product name, category and unit cost come only from the inventory file.
- **Flag, do not change:** odd categories (for example `Sports Socks` under Footwear), abbreviated customer names (`Priya M`), and anything that needs a business decision.
- Margins are stored as fractions and shown as percentages (0.6902 shows as 69.02%).
- Weeks are days 1-7, 8-14, 15-21, 22-28 and 29 to month end (Week 5 only when there are orders on those days).

## Working notes
- Files move between the cloud workspace and the user's computer with tools: copy inputs in with `device_stage_files`, write results to the workspace output folder, then copy them back with `device_commit_files`. A file that is not committed never reaches the user's disk. If a save is blocked because the file is open in another app, tell the user to close it.
- Files on the user's computer cannot be deleted without a shell on that computer. If a rename is asked for, create the new file and tell the user to delete the old one by hand.
- Build the Excel file with the `anthropic-skills:xlsx` skill, the dashboard with the `dataviz` skill and the slide with the `anthropic-skills:pptx` skill. Render the dashboard and the slide to an image and look at them before delivering.
- After each step, add a line to the History section of its step file.
- Reply in plain language and keep it short. Do not recap the steps.

## How to run a new month
1. Put the month's two raw files in `input\` named `sales_<month>_<year>.csv` and `inventory_costs_<month>_<year>.csv`.
2. Send this prompt (replace the month and year):

   `Run the monthly sales pipeline for <Month> <Year>. Follow CLAUDE.md in this project folder: Steps 1, 2 and 3 in order, then Steps 4-1 and 4-2 in parallel. Validate and reconcile to the source data, keep the input files unchanged and tell me what you produced.`

3. If you leave out the month, Claude uses the single ready month that has no outputs yet, or asks which one when there are several.
