# Contract Business KPI Dashboard (Power BI)

An interactive Power BI dashboard for analyzing a B2B contract business: active contracts, monthly and cumulative revenue, customer and product breakdowns, plan-vs-actual performance, and new-signings vs. cancellations — built on a star schema with fully custom DAX.

> **Note on data:** the dataset used in this dashboard is synthetic. It was generated with AI to mirror the structure and scale of a real contract-management dataset I worked with during an internship, without exposing any confidential company data. All numbers are fictional.

(Pictures are following)

## Overview

This dashboard was built as a portfolio project to demonstrate an end-to-end BI workflow: data modeling, DAX measure design, and report/UX design in Power BI. It simulates a recurring-revenue contract business (subscriptions/service contracts) and answers typical management questions:

- How many contracts are currently active, and how is that trending?
- What does monthly and cumulative revenue look like, broken down by customer, product, and business unit?
- How does actual performance compare to plan/forecast?
- How many contracts were newly signed vs. cancelled in a given period?

The report consists of **7 pages**:

| Page | Purpose |
|---|---|
| Übersicht | Comparison of actual data to planning data with KPIs and charts |
| Kundendaten | Revenue and contract counts by customer|
| Analysebaum (Kunden) | Decomposition tree for ad-hoc drill-down by customer dimensions |
| Produktdaten | Revenue and contract counts by product/business unit |
| Analysebaum (Produkt) | Decomposition tree for ad-hoc drill-down by product dimensions |
| Neuabschlüsse/Kündigungen | New signings vs. cancellations |
| Analysebaum (N/K) | Decomposition tree for signings/cancellations |

Every page uses dynamic titles, slicers, and button-based navigation.

![Navigation-Concept](image.png)

## Data Model

The model follows a star schema:

- **`Verträge`** (fact table) — one row per contract, with start/end dates, cancellation date, customer key, and product key
- **`dim_Kundendaten`** — customer master data (customer, country)
- **`dim_Produkt`** — product master data (product, business unit, contract type, monthly/annual price)
- **`Kalender`** — date table
- **`Plan`** — plan/forecast figures by year and business unit


```
dim_Kundendaten  1 ─── M  Verträge  M ─── 1  dim_Produkt
                                              │
                                              M
                                              │
                                              M
                                            Plan
```

`Verträge` connects to both dimension tables via many-to-one relationships. `dim_Produkt` and `Plan` are linked through a many-to-many relationship on `Geschäftsfeld` (business unit) with bidirectional filtering, so plan figures can be filtered consistently alongside actuals from either direction.

## DAX Highlights

A few measures that involved the most interesting modeling decisions:

**Active contracts (events-in-progress pattern)** — a contract counts as active in a selected period if it started before the period ends and hasn't ended (or hasn't ended yet) before the period starts:

```dax
Msr_Aktive_Verträge =
VAR ZeitraumStart = MIN(Kalender[Datum])
VAR ZeitraumEnde = MAX(Kalender[Datum])
RETURN
CALCULATE(
    DISTINCTCOUNT('Verträge'[Vertrags-ID]),
    REMOVEFILTERS('Kalender'),
    'Verträge'[Startdatum] <= ZeitraumEnde,
    OR(ISBLANK('Verträge'[Enddatum]), 'Verträge'[Enddatum] >= ZeitraumStart)
)
```

**Cumulative revenue** — sums monthly revenue across all year/month combinations up to and including the selected period:

```dax
Msr_Umatz_kumuliert =
SUMX(
    SUMMARIZE('Kalender', 'Kalender'[Jahr], 'Kalender'[Monatsnummer]),
    [Msr_Umsatz_Monatl.]
)
```

**Signings and cancellations from the same fact table** — rather than duplicating the fact table, inactive relationships plus `USERELATIONSHIP` let the same `Verträge` table drive both metrics:

```dax
Msr_Anzahl_unterschrieben =
CALCULATE(
    DISTINCTCOUNT('Verträge'[Vertrags-ID]),
    USERELATIONSHIP('Verträge'[Startdatum], 'Kalender'[Datum])
)

Msr_Anzahl_gekündigt =
CALCULATE(
    DISTINCTCOUNT('Verträge'[Vertrags-ID]),
    USERELATIONSHIP('Verträge'[Gekündigt zum], 'Kalender'[Datum])
)
```

**Conditional formatting driven by a measure** — plan-attainment thresholds returned as hex colors and bound directly to visual formatting:

```dax
Msr_Plan_Umsatz%_Farbe =
VAR RawWert = [Msr_Plan_Umsatz_%]
VAR WertProzent = IF(ABS(RawWert) <= 1, RawWert * 100, RawWert)
RETURN
    SWITCH(
        TRUE(),
        WertProzent < 90, "#D64550",
        WertProzent < 95, "#FFC000",
        "#107C10"
    )
```

**Dynamic chart titles** that react to the active filter context:

```dax
Überschrift_Umsatz_Kunde =
VAR MinD = CALCULATE(MIN('Kalender'[Datum]))
VAR MaxD = CALCULATE(MAX('Kalender'[Datum]))
VAR MinY = YEAR(MinD)
VAR MaxY = YEAR(MaxD)
VAR Kunde = SELECTEDVALUE('dim_Kundendaten'[Kunde])
RETURN
"Umsatz "
    & IF(ISBLANK(Kunde), "pro Kunde ", "von " & Kunde & " ")
    & IF(MinY = MaxY, MinY, MinY & " - " & MaxY)
```

The model contains **40+ measures** in total, organized into a dedicated `Msrs` table (core KPIs) and an `Überschriften` table (dynamic titles), keeping the calculation logic separate from the data tables.

## Tech Stack

- **Power BI Desktop** — data modeling, DAX, report design
- **Power Query (M)** — data cleaning and transformation
- **Python (openpyxl)** — synthetic dataset generation
- **Excel** — raw source files

## Repository Structure

```
├── Dashboard.pbix          # Power BI report
├── data/                   # source Excel files (synthetic)
├── docs/                   # screenshots
└── README.md
```

## Getting Started

1. Clone the repo.
2. Open `Dashboard.pbix` in [Power BI Desktop](https://www.microsoft.com/power-bi/downloads).
3. If prompted, update the data source path to point to the local `data/` folder.

## License

Add a license (e.g., MIT) if you want others to reuse this freely.
