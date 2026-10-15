# Data Quality Contract: Retail Sales Feed

| | |
|---|---|
| **Dataset** | `retail_sales_dataset.csv` (one row per transaction; 9 columns) |
| **Consumer / decision owner** | Retail Sales Manager |
| **Data steward** | Analyst responsible for running and triaging the checks |
| **Source system owner** | Team that produces the extract |
| **Version / date** | v1.0, 15 Oct 2026 |
| **Executable proof** | `retail_data_profile.ipynb` (30 checks, exports `dq_results.csv`) |

## 1. Purpose
The Retail Sales Manager uses 10 KPIs (see `retail_kpi_dictionary.xlsx`) to choose which categories to promote and which age groups to target. This contract states what "trustworthy" means for the data feeding those KPIs, how failures are measured, and who acts when a threshold is crossed.

## 2. Guarantees the data provider makes
1. Every row has all 9 columns populated.
2. `Transaction ID` is unique and is the grain (1 row = 1 transaction).
3. Values stay inside their allowed domain (see section 3).
4. `Total Amount` always equals `Quantity x Price per Unit`.
5. The extract is refreshed daily; the newest transaction is at most 1 day old.

## 3. Thresholds
A check is graded on the share of rows that fail it. **WARN** means publish with a banner. **FAIL** means block the affected KPIs.

| Dimension | Check (IDs in notebook) | WARN when | FAIL when |
|---|---|---|---|
| Completeness | All 9 columns non-blank (CMP-00..09) | any blank | 1% or more blank, or a column is missing |
| Uniqueness | Transaction ID unique (UNQ-01), no fully duplicated rows (UNQ-02) | any duplicate | 0.1% or more duplicated |
| Validity | ID positive integer; date parses as dd-mm-yyyy; Customer ID is `CUST`+digits; Gender in {Male, Female}; Age whole number 18-100; Category in {Beauty, Clothing, Electronics}; Quantity whole number > 0; Price > 0; Total >= 0 (VAL-01..10) | any invalid | 0.5% or more invalid; for future-dated rows 0.1% |
| Consistency | Total = Qty x Price (CON-01); no stray spaces or case drift (CON-04); transaction IDs contiguous (CON-03); latest month is a complete period (CON-05) | any mismatch | 0.5% or more mismatched (CON-03 and CON-05 are WARN-only) |
| Freshness | Newest transaction <= 1 day old for daily KPIs (FRS-01); at least 12 months of history (FRS-02) | FRS-02 short | lag over 2 days (daily KPIs) or over 35 days (monthly KPIs) |

Info-level checks (UNQ-03, CON-02) never block a publish; they document facts the analyst must respect, such as each customer appearing only once.

## 4. Release rule
- Any **FAIL on a High-severity check blocks** the daily KPI publish for every KPI that depends on that check (the dependency list is in the `Required DQ Checks` column of the KPI dictionary).
- WARN results publish with a visible banner naming the check.
- Rows that fail a validity rule are quarantined and excluded from KPIs; the count of excluded rows is shown beside the KPI.

## 5. Escalation
| Situation | Who is told | When | What happens |
|---|---|---|---|
| FAIL (High) | Data steward, then source owner | Alert within 1 hour of the run | Publish blocked; triage in 1 business day; fix or written waiver in 2 business days |
| Freshness FAIL | Source owner, and Retail Sales Manager the same day | Same day | KPI tiles marked STALE; manager decides whether to act on old numbers |
| WARN | Data steward | Weekly review | Logged; three consecutive WARNs trigger a threshold review |
| Waiver to publish over a FAIL | Retail Sales Manager approves | Before publish | Reason and expiry date logged |

## 6. Known limitations in the current extract (as of the 15 Oct 2026 run)
- **Freshness: FAIL.** The newest record is 1 Jan 2024 (1,018 days old). The file is a historical snapshot; the check shows a stalled feed would be caught.
- **Partial period: WARN.** Jan 2024 holds 2 transactions versus about 82 in a typical month, so its MoM growth is suppressed.
- **Customer grain.** Every Customer ID appears once, so `Total Customers` equals `Total Transactions` and repeat-purchase, retention or cohort KPIs cannot be built from this extract.
- **No currency, discount, city or payment-status fields.** The first draft of the data dictionary described such fields; they do not exist in this file.
- **Age 65+ band is empty** (oldest customer is 64).

## 7. Change control
Thresholds and allowed lists live in the first notebook cells and in the `quality_contract` sheet. Any change needs the Retail Sales Manager's approval and a new version number.

## 8. How the contract was tested
`retail_sales_dirty_sample.csv` is a copy of the data with 9 injected defects (duplicate ID, impossible date, future date, case/space drift, under-age value, misspelled category, zero quantity, wrong total, blank Customer ID). Running the notebook with `DQ_DATA_PATH` set to that file raises the expected FAIL/WARN results; see `dq_results_dirty_sample.csv`.
