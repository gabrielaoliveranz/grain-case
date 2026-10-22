# Appendix: Data Consolidation and Modelling Methodology

**Analyst:** Gabriela Olivera, Data Analyst · hello@gabrielaolivera.nz · gabrielaolivera.nz
**Version:** 0.9 · **Date:** 08/10/2026

This appendix holds the detail behind the main methodology document: the data dictionaries (A), the full findings log (B), the full decision log (C) and the version history (D). It contains no client names, figures or files. Sources are referred to by the generic codes F01 to F05.

---

## A. Data dictionaries

### A.1 `dim_product`

Source: F01 (item master list). One row per item. Loaded to the Data Model as a dimension.

| Column | Type | Description |
||||
| `ProductID` | Text | Internal key (P001, P002, ...). Assigned after sorting by `ItemCode`, so it depends on alphabetical order and is not used in business rules. |
| `ItemCode` | Text | Business code for the item. Unique across all rows (checked). Used to match other sources. |
| `ItemName` | Text | Item name as recorded in the accounting system. |
| `SalesDescription` | Text | Description shown on sales documents. |
| `SalesUnitPrice` | Number | Selling price per unit recorded on the item. |
| `ItemType` | Text | Classification added in this project: "Grain product" or "Service". Allows services (e.g. transport) to be filtered in the report without removing them from the totals. |
| `ItemKey` | Text | Matching key built from `ItemCode` with spaces removed and all text in uppercase. Used to relate this table to `fct_sales_by_item`, because codes differ in spacing and capitalisation between sources. |

Query steps (in order): Source, Promote headers, Detect types, Remove empty columns, Set data types, Rename columns, Sort by ItemCode, Add index, Add ProductID, Remove index column, Reorder columns, Add ItemType, Remove single-value columns.

### A.2 `fct_sales_by_item`

Source: F02 (inventory items report, PDF). One row per item, period totals only (no date). Loaded to the Data Model as a fact table.

| Column | Type | Description |
||||
| `ItemCode` | Text | Business code for the item, as shown in the report. Matches `dim_product[ItemCode]` once spacing and capitalisation are standardised (see `ItemKey`). |
| `ItemName` | Text | Item name as shown in the report. Kept to check the match against `dim_product`; can be removed once the relationship is validated. |
| `Sales` | Fixed decimal number | Sales amount for the item over the report period. |
| `ItemKey` | Text | Matching key built from `ItemCode` with spaces removed and all text in uppercase. Relates this table to `dim_product`. |

Query steps (in order): Source, Navigation (table selected from the PDF), Promote headers, Replace dash with null, Set data types, Remove empty columns, Remove report label rows, Rename columns. Control query: `chk_inventory_total` (connection only), holding the total row of the report.


### A.3 `fct_income_expenses`

Source: F03 (annual financial statements, PDF). One row per line item and reporting period. Loaded to the Data Model as a fact table.

| Column | Type | Description |
||||
| `LineItem` | Text | Income or expense line as named in the income and expenses schedule. |
| `Period` | Text | Reporting period the amount belongs to (the current year, or the prior comparison period as labelled in the report). |
| `Amount` | Fixed decimal number | Amount for the line item in that period. Negative values are kept as reported (credits or adjustments). |
| `Category` | Text | Classification added in this project: "Income", "Expense" or "Subtotal". Subtotal rows are kept to reconcile against the source and are excluded from totals to avoid double counting. |

Query steps (in order): the three pages of the schedule are appended, then blank rows, section headings and totals are removed, labels are cleaned, values shifted by one column on some rows are realigned, data types are set, errors are replaced with null, and the period columns are unpivoted into rows. Control query: `chk_profit_and_loss` (connection only), used to reconcile totals against the profit and loss statement.

### A.4 `fct_pickups`

Source: F04 (purchases record, image). One row per load picked up. Loaded to the Data Model as a fact table.

| Column | Type | Description |
||||
| `LoadID` | Whole number | Internal key assigned to each load (1, 2, 3, ...). |
| `Date` | Date | Date of the load. The time and time zone that came with the source were removed. |
| `Check Weights` | Decimal number | Weight recorded for the load. Unit to be confirmed with the business (see H03 and H06). |
| `TransportType` | Text | Type of transport used. |
| `PickupFrom` | Text | Place the load was picked up from. Not confirmed as the supplier. |

Query steps (in order): load from the Excel table, remove blank rows, trim text, remove redundant columns, convert `Date` to a date type, set `Check Weights` as a decimal number using the New Zealand locale, add `LoadID`.

---

## B. Findings log

*Issues found during the process, described in generic terms.*

| ID | Date | Source | Finding | Cause | Resolution | Step / query | Status |
|---|---|---|---|---|---|---|---|
| H01 | 30/09/2026 | F01 | Several columns are 100% empty | Fields in the source system's export that are not used by the business | Removed before loading to the Data Model (emptiness checked against the entire data set) | `Remove 100% empty columns` | Resolved |
| H02 | 30/09/2026 | F01 | Some columns hold the same value in every row: status, inventory type ("Untracked"), sales account code, and sales tax rate (one common rate, with one item having no rate recorded) | Attributes are uniform across the item list. "Untracked" indicates stock quantities and costs are not tracked in the accounting system. All sales post to a single account | Removed from the Data Model; values recorded here in case a future export differs | `Remove single-value columns` | Resolved |
| H03 | 30/09/2026 | F04 | The purchases source does not identify the product. It records only date, quantities, supplier and transport type | Source captured for purchasing records, not item-level detail | Linked only by supplier and date; transport type kept as a descriptive attribute (candidate dimension). Product-level analysis is not possible from this source. Recommended to the business: record the type of grain on each pickup | Model relationships | Open (recommendation) |
| H04 | 30/09/2026 | F01 | A transport service is recorded as a product, and one product has no sales | The accounting system lists services and products in the same item list | Both kept. Added `ItemType` ("Grain product" or "Service") so the service can be filtered in the report while totals still reconcile to the source | `Add ItemType` | Resolved |
| H05 | 30/09/2026 | F05 | The contacts list mixes many types of suppliers (utilities, telecommunications, fuel, hardware retail, etc.) with no field identifying the role of each contact. It also holds personal contact details | The list is a general address book of the accounting system, not a purchasing register | Excluded from the Data Model. To be revisited only if a later source needs it to standardise supplier names | n/a | Resolved |
| H06 | 30/09/2026 | F04 | The total written on the purchases record differs slightly from the sum of the individual entries | Likely a manual addition error on the record | Data entered as recorded, without adjusting figures to match. Difference noted in the report | Data entry check | Closed (noted) |
| H07 | 30/09/2026 | F02 | The PDF report mixes structural rows (group label, balance labels, total row) with item rows, and shows zero as a dash | Report layout, not a data table | Table selected in the PDF connector; dashes converted to null; label rows removed; total row kept separately as a control query | `Replace dash with null`, `Remove report label rows` | Resolved |
| H08 | 30/09/2026 | F02 | The five stock movement columns (opening balance, purchases, cost of goods sold, adjustments, closing balance) contain no values, only nulls or dashes. Consistent with items not being stock-tracked (see H02) | Stock is not tracked in the accounting system | Columns removed. No stock analysis is possible from this source; sales by item is retained | `Remove empty columns` | Resolved |
| H09 | 30/09/2026 | F03 | The annual report PDF contains many tables (financial statements and notes), most of which do not relate to the analysis. Some hold sensitive information (related parties, shareholder accounts, tax) | Standard structure of annual financial statements | Only the income and expenses schedule is loaded. The profit and loss statement is kept as a control. All other tables are excluded | Table selection | Resolved |
| H10 | 30/09/2026 | F03 | The income and expenses schedule is split across three tables (one per page) with different column names | The PDF connector reads each page separately | The three tables were appended into one and cleaned once | `Append Queries` | Resolved |
| H11 | 30/09/2026 | F03 | The schedule mixes line items with section headings, totals and status labels (e.g. "DRAFT"). Two lines share the same name, and on some rows the amounts appear one column to the right of where they belong | PDF layout and report formatting | Headings, totals and blank rows removed; labels cleaned; the duplicated name made unique; amounts realigned on the affected rows | Cleaning steps in `fct_income_expenses` | Resolved |
| H12 | 30/09/2026 | F03 | One expense line shows a negative amount | Likely a credit, refund or adjustment recorded in the accounts | Kept as reported so totals reconcile to the control. Can be confirmed with the business | n/a | Open |
| H13 | 30/09/2026 | F04 | The purchases record is an image. Automatic reading returned many low-confidence cells | Image quality and handwriting or layout | Transcribed manually into an Excel table and checked against the image | Data entry | Resolved |
| H14 | 30/09/2026 | F04 | The record has redundant columns (day, year and notes) and the date carries a time and time zone that add nothing | Format of the source | Redundant columns removed and the date converted to a plain date, read with the New Zealand locale so day and month are not swapped | Cleaning steps in `fct_pickups` | Resolved |
| H15 | 08/10/2026 | F02 | The inventory items report has period totals only, with no dates | Report design (summary by item) | Sales by product cannot be filtered by period. Charts built on it are labelled as covering the whole report period | n/a | Resolved |
| H16 | 08/10/2026 | F03 | Start-up and investment costs are mixed with operating expenses in the income and expenses schedule | The accounting report does not separate them | Comparisons between periods carry a note. Recommended to the business: a separate category for these costs | n/a | Open (recommendation) |
| H17 | 08/10/2026 | Report | Charts copied or moved between workbooks lost their connection to the PivotTables and no longer responded to slicers | Static charts keep their values and are not tied to a PivotTable | Charts rebuilt as PivotCharts. Slicers connected to each PivotTable through Report Connections | Report | Resolved |
| H18 | 08/10/2026 | F01, F02 | Item codes are written with different spacing and capitalisation in the item list and the inventory report, so they do not match directly | Each source formats the codes in its own way | `ItemKey` created in both tables (spaces removed, text in uppercase) and used for the relationship. `ItemCode` is kept unchanged | `Add ItemKey` | Resolved |

---

## C. Decision log

*Design decisions and their rationale.*

| ID | Date | Decision | Alternatives considered | Rationale |
|---|---|---|---|---|
| D01 | 30/09/2026 | Layered architecture (staging → model → report) | Performing all transformations in a single query per source | Easier maintenance, traceability and troubleshooting. |
| D02 | 30/09/2026 | Star schema in the Data Model | Single flat table | Avoids duplication, allows sources to be combined and keeps DAX measures simple. |
| D03 | 30/09/2026 | Do not infer missing products; sources are related only through keys that exist in the data | Matching products by quantity, date or supplier | Inferred values cannot be defended or audited. Product-level analysis waits for a source that identifies the product. |
| D04 | 30/09/2026 | Create a numeric-style internal key `ProductID` (P001, P002, ...) and keep `ItemCode` as the business key | Use the item code alone as the key | Stable, lightweight key for relationships, protected from spelling variations. Documented limitation: numbering follows alphabetical order. |
| D05 | 30/09/2026 | Classify the transport service with `ItemType` instead of deleting it | Delete the row | Keeps totals reconcilable to the source and leaves the choice of including or excluding it to the report. |
| D06 | 30/09/2026 | Keep items with no sales in `dim_product` | Remove items without sales | The dimension describes the whole catalogue; an item with no sales is itself a finding. |
| D07 | 30/09/2026 | Exclude the contacts list (F05) from the Data Model | Load it as `dim_supplier` | Contacts have no role identifier and cover unrelated supplier types. It holds personal details that add nothing to the analysis. A customer dimension is not needed for this analysis either. |
| D08 | 30/09/2026 | Keep the report's total row in a separate control query (`chk_inventory_total`) and exclude it from the item table | Leave the total row in the item table | Prevents double counting when summing, and gives an independent figure to reconcile against. |
| D09 | 30/09/2026 | Load only the income and expenses schedule from the annual report and exclude the other statements and notes | Load all tables | The other tables do not link to products, dates or suppliers, and some contain sensitive details about related parties, shareholders and tax. |
| D10 | 30/09/2026 | Use the detailed schedule as the data and the profit and loss statement only as a control | Load both | Both show the same information at different levels of detail. Loading both would double count when summing. |
| D11 | 30/09/2026 | Unpivot the period columns into a `Period` column | Keep one column per year | Allows filtering by period with a slicer and keeps the model consistent with the other fact tables. |
| D12 | 30/09/2026 | Transcribe the purchases image manually rather than accept automatic extraction | Accept the automatic reading and correct only the low-confidence cells | Every value has to be checked against the image anyway, and manual entry gives full control over the result. |
| D13 | 30/09/2026 | Keep weights exactly as recorded (three decimals) and confirm the unit with the business | Assume the unit | Avoids mixing units in totals. The column name will state the unit once confirmed. |
| D14 | 08/10/2026 | Take sales for a period from the Income lines of the income and expenses schedule; take sales by product from the inventory report | Use the inventory report for everything | The inventory report has no dates, so it cannot be filtered by period (see H15). |
| D15 | 08/10/2026 | Use tonnes as the main measure for pickups, with the pickup count as a secondary view | Count of pickups only | Weight reflects real volume; the count does not show differences in load size. |
| D16 | 08/10/2026 | Keep Subtotal rows as a `Category` value and exclude them from totals | Delete subtotal rows | Keeps the source reconcilable while avoiding double counting. |
| D17 | 08/10/2026 | Relate `dim_product` and `fct_sales_by_item` on a standardised `ItemKey`, and keep `ItemCode` unchanged | Edit `ItemCode` directly in each table | The original codes stay available for reporting, and the key is rebuilt the same way in every refresh. |

---

## D. Version history

| Version | Date | Changes |
|---|---|---|
| 0.1 | 30/09/2026 | Initial document structure. |
| 0.2 | 30/09/2026 | Added findings H01 to H03, decision D03, and updated sources F01 and F04. |
| 0.3 | 30/09/2026 | Added `dim_product` (data dictionary and query steps), findings H02 (extended) and H04, decisions D04 to D06, and first quality check result. |
| 0.4 | 30/09/2026 | Added source F05 (contacts list, excluded), finding H05 and decision D07. |
| 0.5 | 30/09/2026 | Added source F02 (inventory items report) with data dictionary for `fct_sales_by_item`, findings H06 to H08, decision D08 and control checks for F02. |
| 0.6 | 30/09/2026 | Updated status of F02 and F04. Added source F03 (annual financial statements) with `fct_income_expenses`, findings H09 to H12, decisions D09 to D11 and control check for F03. |
| 0.7 | 30/09/2026 | Added `fct_pickups` (F04 loaded) with data dictionary, findings H13 and H14, decisions D12 and D13, and F04 control checks. |
| 0.8 | 08/10/2026 | Added `Category` to the `fct_income_expenses` data dictionary and completed the metrics table in section 9. Updated quality checks, H03 and H06. Added findings H15 to H18, decisions D14 to D17 and `ItemKey` to the data dictionaries. |
| 0.9 | 08/10/2026 | Main document shortened. Data dictionaries, findings log, decision log and version history moved to this appendix. No change to their content. |
