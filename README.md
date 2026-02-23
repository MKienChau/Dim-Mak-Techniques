# The Dyson Sphere: Dynamic Measure Engine

> A single dispatcher measure that routes all base metrics through one calculation engine — enabling dynamic visuals, consistent formatting, and zero-redundancy time intelligence across an entire Power BI report.

---

## Design Intent

The core problem this solves: in a typical Power BI report, if you have 20 base metrics, you end up with 20 versions of every derived calculation — 20 YoY measures, 20 prior year measures, 20 format strings, 20 conditional formatting rules. The report becomes brittle and expensive to maintain.

The Dyson Sphere collapses all of that into a single measure: `[Calc]`. Every visual, every derived measure, every format expression references `[Calc]` — not the individual base measures. Switching metrics via a slicer updates everything simultaneously, automatically.

The name reflects the concept: a Dyson Sphere captures all energy output from a star in one structure. Here, one measure captures all metric output from the model.

---

## Architecture Overview

```
Calc[Base] Slicer
       │
       ▼
  ┌─────────┐
  │  [Calc] │  ◄── dispatcher: routes slicer selection to the correct base measure
  └────┬────┘
       │
       ├──► [PY]              Prior year equivalent of [Calc]
       ├──► [YoY]             Absolute year-over-year change
       ├──► [YoY%]            Percentage year-over-year change
       ├──► [YoYperformance]  Relative performance vs peer group
       ├──► [3Y-CAGR%]        3-year compound annual growth rate
       └──► [CY/GT%]          Share of current row vs hierarchy parent total

Format String Expression
       │
       ├── [sv_CalcBase]      Active metric name string
       └── [FLGrpA]           Currency-aware format array
              │
              └── Flag_Currency[Symbol] + Flag_Fmt[GrpA]
```

---

## Component Walkthrough

### [1] `[Calc]` — The Dispatcher

`SWITCH(SELECTEDVALUE(Calc[Base], "NetSls$"), ...)`

Reads the slicer and returns the matching base measure. The default `"NetSls$"` ensures visuals are never blank on load.

**Why not use field parameters directly?**
Field parameters swap column references — they can't reference measures with complex DAX logic. `SWITCH(SELECTEDVALUE(...))` gives full control over which measure is evaluated, without limitation.

**Naming convention on base measures:**
Base measures follow a suffix pattern that encodes their data type:
- `_MeasureName$` → currency
- `_MeasureName%` → percentage
- `_MeasureNameCT` → count/integer

This convention is later exploited by the Format String Expression to infer the correct number format automatically.

---

### [PY] — Prior Year

`CALCULATE([Calc], REMOVEFILTERS(DateFilter[Year]), SAMEPERIODLASTYEAR(CalendarDates[Date]))`

Calculates the prior year value of whatever `[Calc]` is currently returning.

`REMOVEFILTERS(DateFilter[Year])` is required because the report uses a separate year slicer. Without removing that filter first, `SAMEPERIODLASTYEAR` would attempt to shift within an already-filtered year and return blank or incorrect results.

---

### [CY/GT%] — Current vs Grand Total Percentage

Context-aware share calculation. Uses `ISINSCOPE` to detect which level of a hierarchy is currently rendering, then removes filters at that level to compute the correct denominator (parent total).

Evaluation order matters — more granular scopes must appear before broader ones:

| Scope in View | Parent Used for Denominator |
|---|---|
| Individual Customer | Account Type total |
| Account Type Code | Account Type total |
| Account Type | All Customers |
| State/Province | All States (Region total) |
| Region | Country total |
| Country | All Locations |
| Product | Product Type total |
| Product Type | Goods Category total |
| Goods | All Products |

---

### [YoY] and [YoY%]

`[YoY]` returns `BLANK` (not zero) when either `[Calc]` or `[PY]` is blank. This distinction matters — zero implies no change, blank implies no basis for comparison.

`[YoY%]` uses `ABS([PY])` as the denominator. If a prior year value is negative (e.g. a loss metric), a naive divide would flip the sign of the percentage. `ABS` preserves the correct direction of change.

---

### [YoYperformance] — Relative Performance

Compares a row's `[YoY%]` against its peer group average using `ALLSELECTED` at the same hierarchy level. The result is scaled to a ±100 range for consistent conditional formatting regardless of metric magnitude.

Useful for heat maps or color bands where you want to highlight which regions or products are outperforming their cohort — not just the overall average.

---

### [3Y-CAGR%] — 3-Year Compound Annual Growth Rate

Standard CAGR formula: `(EndValue / StartValue) ^ (1 / Years) - 1`

Start year behavior is context-sensitive:
- **Row level (year in scope):** `StartYr = EndYr - 2` → rolling 3-year window per row
- **Total row (no year in scope):** `StartYr = MIN year in dataset` → full data range CAGR

Guard conditions prevent errors when start equals end year, or when no data exists for the start year.

---

### [2] Currency Table (M Query)

A static lookup table in Power Query with columns for currency code, symbol, and description. The selected symbol is injected into the format string array at runtime.

Extension path: replace the static table with a live exchange rate API call to support actual currency conversion.

---

### [3] Format String Expression

Dynamically resolves the correct number format based on the active metric and currency selection — no hardcoded format strings in visuals or individual measures.

**How format is determined:**

The measure name string is parsed using the suffix naming convention:

| Condition | Format Applied |
|---|---|
| Name contains `vM*Pct`, `vM*%`, `ΔM*Pct`, `ΔM*%` | `"0.00"` (fixed decimal, variance measures) |
| Name contains `%` or `pct` | Percentage format from `[FLGrpA]` slot 1 |
| Name contains `unit`, `Qty`, `Quantity`, ends in `CT`, contains `CNT` or `Count` | Unit/count format from `[FLGrpA]` slot 3 |
| Default | Currency format from `[FLGrpA]` slot 2 |

`Flag_Fmt[GrpA]` stores a PATHITEM-encoded string (e.g. `"0.0%|$#,0|#,0"`) where the `$` placeholder is replaced with the selected currency symbol at runtime via `SUBSTITUTE`.

---

## Integration Steps

1. Create base measures with suffix naming (`_NetSls$`, `_Margin%`, `_OrderCT`)
2. Create `Calc` field parameter table — one row per measure name string
3. Create `Flag_Currency` table (currency code, symbol, description)
4. Create `Flag_Fmt` table with PATHITEM-encoded format string arrays
5. Add `[Calc]` dispatcher measure
6. Add derived measures (`[PY]`, `[YoY]`, `[YoY%]`, `[YoYperformance]`, `[3Y-CAGR%]`, `[CY/GT%]`)
7. Add Format String Expression measures (`[sv_CalcBase]`, `[FLGrpA]`, format resolver)
8. Assign Format String Expression to `[Calc]` in the measure properties

---

## Known Considerations

- `SWITCH(SELECTEDVALUE(...))` evaluates the matched branch only — no unnecessary measure evaluation
- `ISINSCOPE` checks must be ordered granular → broad; incorrect order produces wrong parent totals
- The format string expression relies on consistent measure naming — deviating from the suffix convention requires updating the format resolver
- `Flag_Fmt[GrpA]` PATHITEM arrays must maintain slot order: `[1] Pct | [2] Currency | [3] Unit`
