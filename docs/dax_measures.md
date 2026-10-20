# DAX Measures and Model Documentation

**Project:** E-Commerce Sales Analytics UK  
**Author:** Tomisin Obijole

## Scope

This document defines the measures used for the project's analytical
design. Confirm that the expressions match the final PBIX whenever
the report implementation changes.

The model measures recorded positive invoice-line values. It does not
calculate net revenue after returns or profit.

## Model Relationships

| One side | Many side | Configuration |
|---|---|---|
| DimDate[Date] | FactSales[OrderDate] | Active, one-to-many, single direction |
| DimProduct[StockCode] | FactSales[StockCode] | Active, one-to-many, single direction |

InvoiceNo and StockCode must remain Text.

DimProduct uses normalised stock codes and contains one row per product
key. It provides representative descriptions and full-period portfolio
attributes.

Non-merchandise codes without a matching product-dimension row may
appear under a blank product member. Hide that member in the portfolio
slicer rather than deleting fact records.

## Measure Conventions

- Measures recalculate within the current filter context.
- KEEPFILTERS intersects metric eligibility with existing selections.
- DIVIDE returns blank for a zero denominator unless otherwise specified.
- Distinct invoice/customer/product counts are not additive across groups.
- Format monetary measures for display; do not round each source line
  before summation.
- Baselines below assume all report filters and cross-selections are cleared.

## 1. Recorded Positive Value

Sum of recorded positive invoice-line values, including separately
classified charges and accounting entries.

```dax
Recorded Positive Value =
SUM ( FactSales[LineSales] )
```

**Format:** GBP currency, two decimals.  
**Baseline:** £10,642,110.80.

## 2. Merchandise Sales

Recorded value restricted to named-merchandise lines.

```dax
Merchandise Sales =
CALCULATE (
    [Recorded Positive Value],
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** GBP currency, two decimals.  
**Baseline:** £10,255,019.18.

## 3. Positive Invoices

Distinct invoice identifiers represented by retained positive lines.

```dax
Positive Invoices =
DISTINCTCOUNT ( FactSales[InvoiceNo] )
```

**Format:** Whole number.  
**Baseline:** 19,960.

## 4. Value per Invoice

Recorded positive value divided by distinct positive invoices in the
same selection.

```dax
Value per Invoice =
DIVIDE (
    [Recorded Positive Value],
    [Positive Invoices]
)
```

**Format:** GBP currency, two decimals.  
**Baseline:** £533.17.

This is not the average invoice-line value.

## 5. Invoice Lines

Number of retained invoice-line records.

```dax
Invoice Lines =
COUNTROWS ( FactSales )
```

**Format:** Whole number.  
**Baseline:** 524,878.

## 6. Identified Customers

Distinct nonblank CustomerIDs represented in the current selection.

```dax
Identified Customers =
CALCULATE (
    DISTINCTCOUNT ( FactSales[CustomerID] ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** Whole number.  
**Baseline:** 4,338.

## 7. Identified Customer Value

Recorded positive value associated with a nonblank CustomerID.

```dax
Identified Customer Value =
CALCULATE (
    [Recorded Positive Value],
    KEEPFILTERS ( NOT ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** GBP currency, two decimals.  
**Baseline:** £8,887,208.89.

This includes identified-customer non-merchandise entries.

## 8. Unidentified Customer Value

Recorded positive value associated with missing CustomerID.

```dax
Unidentified Customer Value =
CALCULATE (
    [Recorded Positive Value],
    KEEPFILTERS ( ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** GBP currency, two decimals.  
**Baseline:** Approximately £1.75 million.

Missing IDs represent unassigned value, not one anonymous customer.

## 9. Unidentified Value Share

Unidentified-customer value as a share of all recorded positive value
within the current selection.

```dax
Unidentified Value Share =
DIVIDE (
    [Unidentified Customer Value],
    [Recorded Positive Value]
)
```

**Format:** Percentage, one decimal.  
**Baseline:** Approximately 16.5%.

## 10. Merchandise Units

Quantity on retained named-merchandise lines.

```dax
Merchandise Units =
CALCULATE (
    SUM ( FactSales[Quantity] ),
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** Whole number.  
**Baseline:** 5,561,564.

The investigated large positive lines remain included unless the
review-status selection excludes them.

## 11. Merchandise Invoices

Distinct invoices containing retained named-merchandise lines.

```dax
Merchandise Invoices =
CALCULATE (
    [Positive Invoices],
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** Whole number.  
**Validation:** Compare with distinct InvoiceNo in the refreshed
merchandise dataset. A rounded 20K card is not an exact benchmark.

## 12. Active Merchandise Products

Distinct merchandise stock codes present in the selected fact records.

```dax
Active Merchandise Products =
CALCULATE (
    DISTINCTCOUNT ( FactSales[StockCode] ),
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** Whole number.  
**Validation:** Compare with the refreshed, case-normalised merchandise
dataset. Do not reuse the earlier pre-normalisation product count.

Counting from FactSales makes this measure responsive to date and
country filters.

## 13. Identified Invoices

Distinct invoices represented by identified-customer records.

```dax
Identified Invoices =
CALCULATE (
    [Positive Invoices],
    KEEPFILTERS ( NOT ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** Whole number.  
**Validation example:** Netherlands, full period: 94.

## 14. Identified Value per Invoice

Identified-customer value divided by identified invoices.

```dax
Identified Value per Invoice =
DIVIDE (
    [Identified Customer Value],
    [Identified Invoices]
)
```

**Format:** GBP currency, two decimals.  
**Validation example:** Netherlands, full period: £3,036.66.

Use this measure for the identified-customer international comparison.
Its numerator and denominator describe the same subset.

## 15. Top 5 Customer Value Share

Share of selected identified-customer value contributed by the five
highest-value identified customers in that selection.

```dax
Top 5 Customer Value Share =
VAR CustomerValues =
    FILTER (
        ADDCOLUMNS (
            VALUES ( FactSales[CustomerID] ),
            "@Value", [Identified Customer Value]
        ),
        NOT ISBLANK ( FactSales[CustomerID] )
    )
VAR TopCustomers =
    TOPN (
        5,
        CustomerValues,
        [@Value], DESC,
        FactSales[CustomerID], ASC
    )
RETURN
    DIVIDE (
        SUMX ( TopCustomers, [@Value] ),
        [Identified Customer Value]
    )
```

**Format:** Percentage, one decimal.  
**Baseline:** 11.8%.

CustomerID provides a deterministic tie-breaker.

Use this on a summary card. A customer-specific selection narrows the
ranking population. On a table row containing one customer, this measure
can return 100%; it is not that customer's share of a wider population.

## Calculated Columns

### ReversalReviewStatus

Create in FactSales:

```dax
ReversalReviewStatus =
IF (
    FactSales[InvoiceNo] IN { "581483", "541431" },
    "Known matching negative entry",
    "Other positive lines"
)
```

This flags the two investigated positive invoices only.

| Positive invoice | Matching negative invoice | Time difference |
|---|---|---:|
| 581483 | C581484 | 12 minutes |
| 541431 | C541433 | 16 minutes |

Matching records share the customer, stock code, unit price, and
opposite quantity.

The combined positive value is £245,653.20. This is strong evidence
of reversal, not an explicit source linkage or automated matching
of all returns.

Default slicer selection: All.

Selecting Other positive lines is a sensitivity comparison, not
net sales.

### CustomerLabel

Create in FactSales when CustomerID is stored as Whole Number:

```dax
CustomerLabel =
IF (
    ISBLANK ( FactSales[CustomerID] ),
    BLANK (),
    FORMAT ( FactSales[CustomerID], "0" )
)
```

Use this text field on customer chart axes. Exclude blanks and apply
Top 10 by Identified Customer Value.

## Date Table

```dax
DimDate =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2010, 1, 1 ), DATE ( 2011, 12, 31 ) ),
    "Year", YEAR ( [Date] ),
    "MonthNumber", MONTH ( [Date] ),
    "MonthName", FORMAT ( [Date], "MMM" ),
    "YearMonth", FORMAT ( [Date], "yyyy-MM" ),
    "YearMonthSort", YEAR ( [Date] ) * 100 + MONTH ( [Date] )
)
```

Configuration:

1. Mark DimDate as a date table using Date.
2. Sort YearMonth by YearMonthSort.
3. Sort MonthName by MonthNumber.
4. Relate Date to FactSales[OrderDate].
5. Set default slicer coverage to 1 December 2010–9 December 2011.
6. Exclude empty calendar months from monthly value charts.

December 2011 is partial. Its total is not a comparable full-month
decline from November.

## Power Query Product Label

Create in DimProduct as a Custom Column:

```powerquery
[StockCode] & " | " & [Description]
```

This is Power Query M, not DAX.

## Product Portfolio Attributes

These fields originate from full-period Python aggregation:

- RelativePosition
- Rare_5_Invoices
- Low_Sales_100
- Low_Units_50

The screening thresholds are:

- Rare: at most five distinct invoices.
- Low sales: below £100.
- Low units: below 50 units.

RelativePosition uses full-period median sales and unit thresholds.

These attributes are descriptive screens, not profitability measures
or automatic discontinuation rules. They remain fixed when report
filters change.

## Baseline Validation

Clear filters and cross-selections.

| Measure | Expected |
|---|---:|
| Recorded Positive Value | £10,642,110.80 |
| Merchandise Sales | £10,255,019.18 |
| Positive Invoices | 19,960 |
| Value per Invoice | £533.17 |
| Invoice Lines | 524,878 |
| Identified Customers | 4,338 |
| Identified Customer Value | £8,887,208.89 |
| Merchandise Units | 5,561,564 |
| Top 5 Customer Value Share | 11.8% |

Use unrounded source totals during validation. Separately rounded
subtotals may differ by a penny when added.

## Review-Status Validation

Select Other positive lines with all countries and the full period:

| Measure | Expected |
|---|---:|
| Recorded Positive Value | £10,396,457.60 |
| Merchandise Sales | £10,009,365.98 |

These exclude only the two investigated positive invoices. They are
not return-adjusted net-sales figures.

## Refresh Procedure

1. Run `notebooks/E-Commerce_Sales_Analytics_UK.ipynb`.
2. Regenerate the CSVs in `data/processed/`.
3. Update Power Query source paths where necessary.
4. Refresh FactSales and DimProduct together.
5. Check unique product keys and the model relationships.
6. Reconcile the measures above.
7. Restore default filters and save
   `report/E-commerce_Performance.pbix`.