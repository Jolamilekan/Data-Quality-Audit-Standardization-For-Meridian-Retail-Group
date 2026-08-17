# Data Quality Audit & Standardization — Meridian Retail Group

## Overview

Meridian Retail Group operates across web, mobile, and marketplace channels, with customer, product, order, returns, and marketing data maintained across five separate systems. Ahead of a planned analytics initiative, Meridian's data was exported from each system for the first time as a combined view — customer records from the CRM, inventory data from the warehouse system, transactions from the web platform, service records from customer support, and campaign data from the marketing team.

Because these systems were never designed to share data, the combined export revealed significant inconsistencies: conflicting customer records, unusable contact information, mismatched currencies and units, and transactions that referenced customers and products no other system could confirm existed. This report documents the audit, the standardization work performed, and the findings that matter for how Meridian's data should be handled going forward.

**Scope:** Customer, Product, Order, Order Line, Returns, and Marketing Campaign records — approximately 29,000 records in total.

---

## Table of Contents

1. [Dataset](#dataset)
2. [Analysis](#analysis)
3. [Executive Summary](#executive-summary)
4. [Recommendations](#recommendations)
5. [Limitations](#limitations)

---

## Dataset

| Record Type | Volume | Source |
|---|---|---|
| Customers | ~3,000 | Customer Relationship Management system |
| Products | 250 | Warehouse / Inventory system |
| Orders | 9,000 | Web & mobile storefront |
| Order Line Items | 16,000 | Web & mobile storefront |
| Returns | 900 | Customer support system |
| Marketing Campaigns | 40 | Marketing team records |

Each system recorded and formatted its data independently, with no shared standard for dates, currency, geography, or identifiers across departments.

---

## Analysis

### Customer Records

The CRM export contained customer records with inconsistent country naming, conflicting duplicate entries under the same customer ID, and a significant share of contact information that could not be relied upon. Age data included values that were not physically possible (negative ages, ages exceeding 150), pointing to either data entry error or an unvalidated form field. Marketing consent status was recorded in at least six different formats across the customer base.

 ![Customer records prior to standardization](Data%20before%20Cleaning/crm_customer_raw%20data.png)

Country names, marketing consent values, and duplicate customer entries were standardized to a single consistent format, with the most recent record retained where duplicates existed. Email and phone numbers found to be structurally invalid were flagged for follow-up rather than altered, since correcting contact information without verified source data risks introducing new errors rather than resolving real ones.

 ![Standardized customer records with data quality flags](Data_screenshots_after_Cleaning/crm_customer_clean%20data.png)

### Product Records

Inventory data mixed at least four currencies within a single price field, with no consistent labeling of which currency applied to which record. Product weight was recorded in kilograms, pounds, and grams interchangeably, and a portion of records had no unit specified at all. A small number of products were recorded with negative stock quantities, a physical impossibility indicating an inventory system error.

  ![Product records prior to standardization](Data%20before%20Cleaning/inventory_products_raw%20data.png)

Prices were standardized to a single currency using documented conversion assumptions (see Limitations). Weight was standardized to kilograms where the recorded unit was identifiable; records with no unit specified were left unresolved rather than guessed, since an incorrect assumption here would materially distort weight-based reporting. Negative stock values were flagged and excluded from stock-level totals.

![Standardized product records with data quality flags](Data_screenshots_after_Cleaning/inventory_product_clean%20data.png)

### Orders

Order dates were recorded in at least six different formats, consistent with the storefront system logging dates differently depending on channel (web, mobile, marketplace, phone). A small number of orders referenced customers that do not exist anywhere in the CRM — likely test transactions or customers whose CRM records were since deleted. A separate, more significant issue was only discovered while validating relationships between tables: a portion of order records shared duplicate order identifiers, which would have caused transactions to be miscounted or misattributed had it gone unaddressed.

 ![Orders records prior to standardization](Data%20before%20Cleaning/orders_export_raw%20data.png)

Dates were standardized to a single consistent format, with a small number of genuinely invalid entries (e.g., non-existent calendar dates) retained as unresolved rather than estimated. Orders referencing unrecognized customers were flagged rather than removed, since the transaction itself remains valid revenue regardless of whether the customer record can be matched. Duplicate order identifiers were resolved prior to finalizing the dataset.

`![Order dates before standardization, showing multiple formats](screenshots/orders_before.png)`

### Order Line Items

Line-level transaction data included a small number of impossible quantities (negative or zero units sold) and one notable outlier quantity warranting manual review rather than automatic correction - 999 stock quantity. Pricing fields mixed currency symbols directly into numeric values, and currency labeling was inconsistent across records. A number of line items referenced products that do not exist in the current inventory system, most likely discontinued or since-renamed products.

 ![Customer records prior to standardization](Data%20before%20Cleaning/orderlines_export_raw%20data.png)

Invalid quantities were flagged and excluded from quantity-based totals; the outlier was flagged for business review rather than treated as an error, since it may represent a legitimate bulk order. Pricing and currency fields were cleaned and standardized. Line-item totals were recalculated from verified quantity and price data rather than trusting the originally recorded total, which had been calculated upstream before any validation occurred.

### Returns

Return records showed the same date-format inconsistency seen in Orders, along with refund amounts recorded as negative values and some prefixed with currency symbols. A number of returns referenced orders that do not exist in the order system. Most significantly, cross-referencing return dates against their associated order dates revealed a subset of returns recorded as occurring *before* the corresponding order was placed — a logical impossibility only visible once the two systems were compared directly.

 ![Returns records prior to standardization](Data%20before%20Cleaning/returns_export_raw%20data.png)

Refund amounts were corrected to reflect actual monetary value, and return reasons were standardized, with unspecified reasons labeled explicitly rather than left blank. Returns referencing unrecognized orders were flagged. The date-sequence issue was documented as a priority finding for the support systems team (see Recommendations).

`![Returns recorded before their associated order date](screenshots/returns_logic_flag.png)`

### Marketing Campaigns

Campaign records mixed duration formats (days, weeks, and unlabeled numbers within the same field) and multiple currencies within the budget field, consistent with the same lack of shared standards seen elsewhere. A portion of impressions data was missing or marked as unavailable rather than recorded as zero.

 ![Marketing records prior to standardization](Data%20before%20Cleaning/campaign_export_raw%20data.png)

Duration was standardized to a single day-count measure, and budget was standardized to a single currency. Missing impressions were preserved as unknown rather than assumed to be zero, since the two carry materially different meaning for campaign performance reporting.

---

## Executive Summary

Meridian's first combined data export surfaced data quality issues that were not visible to any single department working within its own system. Every record type required both formatting correction and cross-system validation — the two are not the same exercise, and formatting fixes alone would have missed several of the most consequential findings.

**What stood out:**

- **Contact data reliability is a concern.** A meaningful share of customer phone numbers, and a smaller share of email addresses, do not meet basic structural validity — meaning a real portion of Meridian's customer base may be unreachable through recorded contact information.
- **No shared data standards exist across systems.** Currency, date format, and unit of measure were each recorded inconsistently across at least three of the five source systems, indicating this is a systemic gap rather than isolated data entry error.
- **Referential gaps exist between systems.** Orders reference customers the CRM doesn't recognize; order line items reference products inventory doesn't recognize; returns reference orders that don't exist. None of these systems currently validate against one another before recording a transaction.
- **The most serious finding was logical, not cosmetic:** a subset of returns are recorded as happening before the order that supposedly generated them — a sequencing error that formatting cleanup alone would never surface, and one worth investigating at the process level, not just the data level.

Following standardization, all six record types were successfully linked into a single connected dataset, confirming the data is now structurally sound enough to support reliable reporting.

---

## Recommendations

- **Introduce a shared date format standard** across systems, or at minimum a standardized export format, to eliminate the current six-plus format variation.
- **Standardize currency handling at the point of entry**, with explicit currency tagging on any price or budget field, rather than relying on mixed, unlabeled values.
- **Add validation on email and phone fields at capture**, given the volume of unusable contact records currently on file.
- **Introduce cross-system checks before finalizing a transaction** — specifically validating that a customer and product referenced in an order actually exist in their respective source systems.
- **Add a sequencing rule preventing a return from being logged with a date earlier than its associated order** — this is a process-level fix, not a data cleanup fix, and would prevent the issue from recurring.

---

## Limitations

- Dates that could plausibly belong to more than one format were resolved using a documented assumption (day-before-month), not verified against the original source system. A small number of records may be affected.
- Currency conversion used a fixed rate rather than a rate tied to each transaction's actual date, as historical exchange rate data was not available.
- Contact information flagged as invalid was left unmodified rather than corrected, since reformatting without confirmed source data risks introducing new inaccuracies.
- Records referencing unrecognized customers, products, or orders were retained rather than removed, as they typically still represent valid transactions — any reporting requiring fully matched records should account for this.
- Product weight values with no recorded unit were left unresolved rather than assumed, to avoid introducing a significant, undetectable error into weight-based figures.
