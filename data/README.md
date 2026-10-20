# Data Sources and Preparation

## Source

[UK Online Retail dataset on Kaggle](https://www.kaggle.com/datasets/carrie1/ecommerce-data).

The source file used in this analysis contains 541,909 invoice-line
records and eight columns:

- InvoiceNo
- StockCode
- Description
- Quantity
- InvoiceDate
- UnitPrice
- CustomerID
- Country

Observed transaction coverage is 1 December 2010–9 December 2011.

## Local Folder Layout

Create these directories locally:

```text
data/
├── README.md
├── raw/
│   └── data.csv
└── processed/
    ├── positive_sales.csv
    ├── named_merchandise_sales.csv
    ├── product_summary.csv
    ├── identified_customer_summary.csv
    └── monthly_summary.csv
```

Raw and processed data files are not required to be committed to GitHub.
Check the original dataset licence before redistributing either.

## Obtain the Source

1. Open the dataset link.
2. Download and extract its CSV.
3. Save the file as `data/raw/data.csv`.
4. Run `notebooks/E-Commerce_Sales_Analytics_UK.ipynb`.

The notebook reads the source using `encoding="latin-1"`.

## Analytical Populations

### Raw records

The original source contains 541,909 invoice lines.

### Positive-value records

After exact-row deduplication, retain records with:

- A valid invoice date.
- Quantity > 0.
- UnitPrice > 0.

Missing CustomerIDs are retained.

Documented baseline: 524,878 invoice lines and £10,642,110.80 in
recorded positive value.

### Named-merchandise records

Apply the additional documented exclusions for postage, fees, accounting
adjustments, ambiguous MANUAL entries, and missing descriptions.

Documented baseline: £10,255,019.18 in named-merchandise sales and
5,561,564 units.

This analytical definition does not guarantee that every remaining code
represents ordinary physical stock; vouchers and samples may require
separate treatment for inventory questions.

### Identified-customer records

Use positive-value records with a nonmissing CustomerID.

Documented baseline: 4,338 identified customers and £8,887,208.89 in
identified-customer positive value.

## Important Cleaning Rules

- Preserve InvoiceNo as text, including `A563185`.
- Preserve StockCode as text.
- Standardise stock-code case before product aggregation.
- Remove exact raw-row duplicate copies under the documented assumption.
- Do not perform another automatic fact-row deduplication merely because
  stock codes have been normalised.
- Do not remove high quantities solely because they are statistical outliers.
- Keep the two investigated positive invoices in the default population.
- Do not replace missing CustomerIDs with an invented customer identity.

## Generated Files

| File | Purpose |
|---|---|
| positive_sales.csv | Invoice-line source for FactSales in Power BI |
| named_merchandise_sales.csv | Merchandise-only analysis and reconciliation |
| product_summary.csv | Product descriptions, full-period groups, and screening flags |
| identified_customer_summary.csv | Customer-level reconciliation |
| monthly_summary.csv | Monthly reconciliation |

The Power BI report uses invoice-line measures rather than summing
pre-aggregated summary totals into the fact table.

## Power BI Import

Import:

- `positive_sales.csv` as `FactSales`.
- `product_summary.csv` as `DimProduct`, retaining descriptive attributes
  and classification flags.

Use the file origin that correctly reads exported text, including the
pound symbol. Pandas CSV exports default to UTF-8 unless configured
otherwise; do not assume the raw file's Latin-1 encoding also applies
to processed exports.

Update Power Query source locations when moving the project.

## Reconciliation

With full coverage and no country, product, customer, or review filter:

| Check | Expected |
|---|---:|
| Invoice lines | 524,878 |
| Recorded positive value | £10,642,110.80 |
| Named-merchandise sales | £10,255,019.18 |
| Positive invoices | 19,960 |
| Identified customers | 4,338 |
| Merchandise units | 5,561,564 |

Product counts must use the refreshed post-normalisation output.
Do not reuse a product count calculated before combining stock-code
case variants.

## Interpretation

Recorded positive value excludes negative records and is not net revenue.
Named-merchandise sales apply a narrower scope. Neither measure represents
profit or cash received.