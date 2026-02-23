# Dealer's Choice: Multi-Dynamic Measure Selector

> A multi-select slicer system that lets users pick any combination of metrics and assign them positionally to visual columns — dynamically, without modifying the report.

---

## Design Intent

The Dyson Sphere handles one metric at a time — the user picks one measure and the entire report updates to reflect it. Dealer's Choice extends that concept to N metrics simultaneously.

The problem it solves: stakeholders often want to compare specific metrics side by side, but the combination changes depending on what they're analyzing. Building a static table with fixed columns forces either too many columns or too many report pages. Dealer's Choice lets the user deal their own hand — pick 2, 3, or 4 metrics from a slicer and they populate the table columns in the exact order selected.

The name reflects the mechanic: the user chooses what's on the table.

---

## Architecture Overview

```
Calc[Overview] Table (multi-select slicer)
       │
       ▼
  [CY_Sel] measure
  Captures slicer state as pipe-delimited string
  "Margin$|NetSls$|OrderCT"
       │
       ▼
  Calc_0 | Calc_1 | Calc_2 | Calc_N  (placed as columns on visual)
       │
       └── SELECTEDMEASURENAME() → parses trailing digit → PATHITEM index
           → resolves which metric string maps to this slot
           → SWITCH routes to the corresponding base measure
```

---

## Component Walkthrough

### Calc[Overview] — Slicer Table

A static DAX table containing one row per metric name string. This is the multi-select slicer the user interacts with. The strings must exactly match the SWITCH keys in the Calc_N dispatcher measures.

Order within the table controls display order in the slicer UI, but selection order is what drives positional assignment to visual columns.

---

### [CY_Sel] — Selection State String

`IF(ISFILTERED(Calc[Overview]), CONCATENATEX(Calc, Calc[Overview], "|"))`

Captures the current multi-select slicer state as a single pipe-delimited string. The pipe delimiter is intentional — it makes the string directly compatible with `PATHITEM` for positional lookup.

The `ISFILTERED` guard returns `BLANK` when nothing is selected, which causes all Calc_N slots to blank out cleanly rather than falling back to a default.

**Example output:**
- User selects Margin$, then NetSls$, then OrderCT
- `[CY_Sel]` = `"Margin$|NetSls$|OrderCT"`

---

### Calc_0, Calc_1, Calc_2, Calc_N — Slot Measures

These are the measures placed as columns on a visual. Each is identical in logic — only the measure name differs, and that name is the key.

**How positional resolution works:**

```
_SMN = SELECTEDMEASURENAME()        -- returns "Calc_1" when evaluated in Calc_1
RIGHT(_SMN, 1)                      -- extracts "1"
INT(RIGHT(_SMN, 1))                 -- converts to integer: 1
PATHLENGTH(_sv) > 0                 -- returns TRUE (1) when selections exist
DIVIDE(1, 1)                        -- passes index through: 1
PATHITEM(_sv, 1, TEXT)              -- retrieves 1st item: "Margin$"
SWITCH("Margin$", ...)              -- routes to [_Margin$]
```

Each slot measure self-identifies via `SELECTEDMEASURENAME()`, extracts its own index from its name, and uses that index to look up its assigned metric from the selection string. No external mapping table, no configuration — the measure name itself is the routing mechanism.

**Slot behavior at a glance:**

| Measure | Index | With selection "Margin$\|NetSls$\|OrderCT" |
|---|---|---|
| Calc_0 | 0 | BLANK (0-index not used / label slot) |
| Calc_1 | 1 | Margin$ |
| Calc_2 | 2 | NetSls$ |
| Calc_3 | 3 | OrderCT |
| Calc_4 | 4 | BLANK (no 4th selection) |

Unused slots return BLANK automatically — no conditional logic needed on the visual side.

---

## Integration Steps

1. Create the `Calc[Overview]` table with metric name strings matching your base measure SWITCH keys
2. Create `[CY_Sel]` in your measures table
3. Create Calc_0 through Calc_N (as many slots as the maximum number of columns you want to support)
4. Place Calc_0 through Calc_N as columns on a matrix or table visual
5. Add a slicer sourced from `Calc[Overview]` — set to **multi-select**
6. Optionally hide unused Calc_N columns via visual-level formatting or conditional visibility

---

## Key Considerations

- **Selection order matters** — the order in which the user clicks slicer items determines which slot each metric lands in. This is intentional but worth communicating to end users.
- **Calc_0 index behavior** — `PATHITEM` is 1-based. A trailing digit of `0` passed to `PATHITEM` returns BLANK, making Calc_0 a natural label or header slot. If you want Calc_0 to carry a value, rename the measure so its trailing digit starts at `1` (e.g. Calc_1 through Calc_N).
- **SELECTEDMEASURENAME() dependency** — this function only returns meaningful values when the measure is evaluated in a visual context where it can identify itself by name. It will not work correctly if the measure is called indirectly from another measure.
- **Slot count is fixed at report design time** — you define how many Calc_N measures exist. Users cannot exceed that number of simultaneous selections meaningfully. Build enough slots for the maximum expected use case.
- **SWITCH keys must match slicer strings exactly** — any mismatch between `Calc[Overview]` values and SWITCH cases results in the fallback default being returned silently.
