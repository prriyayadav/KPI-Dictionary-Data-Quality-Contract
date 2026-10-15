# KPI Dictionary & Data Quality Contract: Retail Sales
 
Turns an ambiguous business request ("how are sales doing, and can I trust the numbers?") into 10 measurable KPIs and a testable agreement about what trustworthy data means.
 
- **Dataset:** `retail_sales_dataset.csv`, 1,000 transactions (1 Jan 2023 to 1 Jan 2024), 3 product categories
- **Decision owner:** Retail Sales Manager (category focus, promotions, age-group targeting)
## Deliverables
 
| File | What it is |
|---|---|
| `retail_kpi_dictionary.xlsx` | KPI dictionary (10 KPIs with formula, grain, filters, owner, refresh cadence, live values), breakdown tables, corrected data dictionary, check results, contract thresholds |
| `retail_data_profile.ipynb` | Executed notebook: 30 checks across completeness, uniqueness, validity, consistency and freshness; PASS / WARN / FAIL grading; publish-or-block decision; KPI computation |
| `data_quality_contract.md` | Guarantees, thresholds, release rule, escalation owners and response times, known limitations |
| `retail_sales_dirty_sample.csv` | Copy of the data with 9 injected defects, used to prove the checks can fail |
| `dq_results_dirty_sample.csv` | Notebook output on the dirty sample |
 
## Workbook sheets
 
| Sheet | Contents |
|---|---|
| `README` | Purpose, key findings, colour code |
| `kpi_dictionary` | 10 KPIs with live formula values |
| `kpi_breakdowns` | Sales by category, age group and month (with MoM growth), plus reconciliation checks that must read OK |
| `sales_data` | Source rows plus helper columns (Month Start, Age Group) |
| `data_dictionary` | Corrected column definitions and rules |
| `dq_checks` | All checks with severity, thresholds and last result |
| `quality_contract` | Thresholds and escalation table |
 
## KPIs
 
1. Total Sales Amount
2. Total Transactions
3. Units Sold
4. Average Order Value
5. Total Customers
6. Sales Amount by Product Category
7. Sales Amount by Age Group
8. Monthly Sales and MoM Growth %
9. Average Units per Transaction
10. Category Share of Sales %
Current values: sales 456,000; 1,000 transactions; 2,514 units; AOV 456.
 
## How to run
 
```bash
pip install pandas numpy matplotlib jupyter
jupyter nbconvert --to notebook --execute --inplace retail_data_profile.ipynb
```
 
The notebook finds `retail_sales_dataset.csv` automatically. To profile a different file:
 
```bash
DQ_DATA_PATH=path/to/file.csv jupyter nbconvert --to notebook --execute --inplace retail_data_profile.ipynb
```
 
Each run writes `dq_results.csv`. Freshness is measured against today's date; change `AS_OF` in the config cell to override. To refresh the `dq_checks` sheet in the workbook, re-run the notebook and paste the new `dq_results.csv`.
 
## Current results (run on 15 Oct 2026)
 
28 PASS, 1 WARN, 1 FAIL.
 
- **Row-level quality is clean:** no nulls or duplicates, and every `Total Amount` equals `Quantity x Price per Unit`.
- **Freshness FAIL:** newest record is 1 Jan 2024 (1,018 days old). The file is a historical snapshot, so this shows the monitor would catch a stalled feed.
- **Partial-month WARN:** Jan 2024 has 2 rows against about 82 in a typical month, so its MoM growth is suppressed.
## Known limitations
 
- Each Customer ID appears once, so Total Customers equals Total Transactions and repeat-purchase or retention KPIs are not possible from this extract.
- There is no currency, discount, city or payment-status field in the data.
- The 65+ age band is empty (oldest customer is 64).
- Thresholds and the 1-day freshness limit are assumptions; change them in the notebook config cell and the `quality_contract` sheet with the owner's approval.
## How the contract was tested
 
The notebook was run on `retail_sales_dirty_sample.csv`, which has a duplicate ID, an impossible date, a future date, case/space drift, an under-age value, a misspelled category, a zero quantity, a wrong total and a blank Customer ID. The expected FAIL and WARN results fired (see `dq_results_dirty_sample.csv`).
 
