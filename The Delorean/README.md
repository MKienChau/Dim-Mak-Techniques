# The Delorean: Dynamic Time Intelligence Slicer

> A fully self-contained, slicer-driven time intelligence system built entirely in DAX — no Power Query date table required, no hardcoded period measures, no report modifications needed when the calendar year rolls over.

---

## Design Intent

Standard time intelligence implementations in Power BI have a fundamental problem: they're static. You build a YTD measure, a prior year measure, a trailing 12-month measure — and each one is hardcoded to its period. Adding a new period type means adding new measures. Changing what "current" means requires touching the model.

The Delorean solves this by inverting the approach. Instead of building one measure per time period, the calendar table itself encodes every time intelligence period as a column. A single slicer lets the user select any period type — ThisYear, QTD, Trailing 90d, LMTD, and 30+ others — and the model responds dynamically without any measure changes.

The name reflects the concept: time travel on demand, returning to any period from a single control.

---

## Architecture Overview

```
CalendarDates (TI-Centric Calendar Table)
       │
       ├── Anchor Variables (TODAY-relative: CurDt, YR, QT, MN, WK, DY)
       ├── Last Completed Period Anchors (LEQ, LEM, LEW / LSQ, LSM, LSW)
       │
       ├── [1] Base Calendar Columns (Year, Quarter, Month, Week, Day, Season)
       │
       └── [2] TI Columns — one per period type, value = "P{YearΔ}" or BLANK
                │
                ├── This(Y/Q/M/W/D)
                ├── Prior(Y/Q/M/W/D)
                ├── TD(Y/Q/M/W)
                ├── LTD(Y/Q/M/W)
                └── Trailing(4Q/2Q/1Q | 12m–2m | 52w–2w | 365d–7d)

DateFilter Table (TI Slicer)
       │
       ├── [2] TI_Tble — INFO.VIEW.COLUMNS() extract for TIGroup/TISort metadata
       ├── [3] RawTable — UNION of SUMMARIZECOLUMNS per TI column
       └── [4] Final Table — ADDCOLUMNS enrichment (Period#, Year, TIGroup, TISort)

Guardrails
       └── Year slicer + TI slicer — both set to single select dropdown
```

---

## Component Walkthrough

### Anchor Variables

All time intelligence logic derives from a single source of truth: `TODAY()`. Anchor variables are computed once at table evaluation time and referenced throughout every column definition.

Two sets of anchors are computed for "last completed" periods:

| Variable | Meaning |
|---|---|
| `LEQ` | Last completed quarter end date |
| `LEM` | Last completed month end date |
| `LEW` | Last completed week end date |
| `LSQ` | Last completed quarter start date |
| `LSM` | Last completed month start date |
| `LSW` | Last completed week start date |

These anchors are what makes Trailing period calculations self-updating — they always resolve relative to today, not a hardcoded date.

---

### [1] TI-Centric Calendar (`CalendarDates`)

Built in two passes using nested `ADDCOLUMNS` on a `CALENDAR(SD, ED)` base:

**Pass 1 — Base calendar columns:** Standard date attributes (Year, Quarter, Month, Week, Day, Season). Multiple format variants per granularity (e.g. `MonthLabel` = "Jan", `MonthYr` = "Jan'25", `MonthsYr` = "01/25") to support different visual labeling needs without calculated columns downstream.

**Pass 2 — TI columns:** One column per time intelligence period. Each column returns either:
- `"P{YearΔ}"` — a string encoding which year offset the date belongs to for that period
- `BLANK` — the date does not fall within that period

The `P{YearΔ}` pattern is the key design decision. `P0` = current year, `P1` = one year ago, `P2` = two years ago. This allows multiple years of comparison data to coexist in the same column — the Year slicer then filters which `P` value is active.

**TI Column Categories:**

| Category | Columns |
|---|---|
| This (current period) | ThisYear, ThisQtr, ThisMonth, ThisWk, ThisDay |
| Prior (last completed period) | PriorYear, PriorQtr, PriorMonth, PriorWk, PriorDay |
| To-Date | YTD, QTD, MTD, WTD |
| Last-To-Date | LYTD, LQTD, LMTD, LWTD |
| Trailing Quarters | Trailing4Q, Trailing2Q, Trailing1Q |
| Trailing Months | Trailing12m, Trailing6m, Trailing3m, Trailing2m |
| Trailing Weeks | Trailing52w, Trailing26w, Trailing12w, Trailing8w, Trailing4w, Trailing2w |
| Trailing Days | Trailing365d, Trailing180d, Trailing120d, Trailing90d, Trailing60d, Trailing30d, Trailing14d, Trailing7d |

**Trailing period logic:**
Trailing columns use `YEARFRAC` to compute `yd` (year offset), then `EDATE` to shift the anchor back by `12*yd` months. This makes trailing windows year-comparable — selecting Trailing12m shows the same window for P0 and P1 side by side when both years are in scope.

---

### [2] TI Details Table (`TI_Tble`)

`INFO.VIEW.COLUMNS()` is used to extract metadata directly from the model — specifically the `DisplayFolder` and `Name` fields for columns in `CalendarDates` that live under the `_TimeIntelligence` folder.

This produces a lookup table with:
- `TimeIntelligence` — column name (e.g. "Trailing12m")
- `TIGroup` — the grouping label parsed from the display folder path (e.g. "TrailingMonths")
- `TISort` — numeric sort order parsed from the display folder path

Why `INFO.VIEW.COLUMNS()` instead of a hardcoded table: adding a new TI column to the calendar and placing it in the correct display folder automatically includes it in the slicer metadata — no manual table updates required.

---

### [3] RawTable — UNION of SUMMARIZECOLUMNS

Each TI column gets its own `SUMMARIZECOLUMNS` call that:
1. Filters `CalendarDates` to only rows where that TI column is not blank
2. Counts total dates in that period (`DateCT`)
3. Tags the row with the TI column name and the active `Period` value

All 38 results are `UNION`'d into a single flat table. This becomes the DateFilter slicer table — one row per TI period per year in scope.

---

### [4] Final Enrichment — ADDCOLUMNS on RawTable

Four derived columns are added to the union result:

| Column | Purpose |
|---|---|
| `Period#` | Numeric period offset — strips "P" from the period string for sorting |
| `CT/Period` | Estimated periods per year — `DateCT / 6` approximates the number of comparable windows |
| `Year` | Resolves the calendar year from the period offset |
| `TIGroup` | Joins TIGroup label from `TI_Tble` via MAXX/FILTER lookup |
| `TISort` | Composite sort key — TISort prefix + DateCT rank suffix, zero-padded for correct string sort order |

The `TISort` composite key is particularly important: it ensures the slicer displays period types in a logical order (grouped by category, then ordered by window size within each category) rather than alphabetically.

---

### [5] Guardrails

Two slicers control the system:
- **Year slicer** — filters which `P{YearΔ}` value is active (single select, dropdown)
- **Time Intelligence slicer** — filters which TI period type is selected (single select, dropdown)

Both must be set to **single select** to prevent ambiguous filter contexts. Multi-select on either would allow combinations that produce undefined or misleading results in downstream measures.

---

## Integration Steps

1. Create the `CalendarDates` calculated table using the DAX in this file
2. Set display folders for all TI columns under `_TimeIntelligence\{GroupName}_{SortOrder}` structure
3. The `TI_Tble` variable reads those folders automatically via `INFO.VIEW.COLUMNS()`
4. Create the `DateFilter` calculated table using the UNION/ADDCOLUMNS structure
5. Add a Year slicer sourced from `DateFilter[Year]`
6. Add a Time Intelligence slicer sourced from `DateFilter[TimeIntelligence]`
7. Set both slicers to single select dropdown
8. Connect `CalendarDates` to your fact table on `Date`
9. In measures, filter using the active TI column — the `DateFilter` table drives which dates are in scope

---

## Known Considerations

- `INFO.VIEW.COLUMNS()` requires display folders to be set correctly — the folder naming convention `_TimeIntelligence\{Group}_{Sort}` must be consistent for `TIGroup` and `TISort` to resolve correctly
- The `P{YearΔ}` encoding means the Year slicer and TI slicer are interdependent — the year slicer selects the offset, the TI slicer selects the period type
- Trailing period windows are anchored to the last *completed* period (LSQ, LSM, LSW) — not today — to avoid partial period distortion
- `CT/Period` (`DateCT / 6`) is an approximation used for display context, not a precise calculation
- Table evaluation happens at model load/refresh — anchor variables resolve at that moment, not at query time
