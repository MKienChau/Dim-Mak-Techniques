# Dim Mak Techniques
### Advanced Power BI & DAX Design Patterns

> A collection of precision-engineered Power BI framework designs. Each technique targets a specific pressure point in report architecture — dynamic measure routing, time intelligence, multi-metric selection, hierarchical ranking, and slicer navigation — to produce disproportionate impact from a small, well-placed implementation.

---

## The Collection

| Design | Type | Description |
|---|---|---|
| [The Dyson Sphere](./dyson-sphere/README.md) | DAX | Single dispatcher measure that routes all base metrics dynamically — one measure powers the entire report |
| [The Delorean](./delorean/README.md) | DAX | Self-updating time intelligence system with 38 period types driven by a single slicer |
| [Dealer's Choice](./dealers-choice/README.md) | DAX | Multi-select slicer that assigns user-chosen metrics to visual columns positionally |
| [Cascading Ranking](./cascading-ranking/README.md) | DAX | Hierarchy-aware ranking using a tier-based attribute system *(in development)* |
| [Yellow Pages](./yellow-pages/README.md) | Design Pattern | Bookmark-driven nested slicer navigation system for 50+ filters |

---

## Design Philosophy

These are not templates. They are techniques — each one addresses a specific architectural problem that conventional Power BI implementations handle poorly or don't handle at all.

The common thread across all of them: **precision over brute force.** Rather than adding more measures, more pages, or more visuals, each design finds the exact point in the model or report where a targeted solution produces the most leverage.

They are also designed to work together. The Dyson Sphere dispatcher feeds into the Delorean's time intelligence. Dealer's Choice extends the dispatcher pattern to multi-metric selection. Cascading Ranking builds on both. Yellow Pages wraps the whole system in a navigable UX layer. Each technique is independently useful but the full collection is greater than the sum of its parts.

---

## Article Series

These designs are documented in an ongoing article series. Each article builds toward the full implementation:

**The Dyson Sphere** — Dynamic Measure Engine
- Article 1: Dynamic Measures
- Article 7: Context-Sensitive Format String Expression
- Article 8: Dynamic Format String Expression
- Article 9: Kardashev Scale: Type II

**The Delorean** — Dynamic Time Intelligence Slicer
- Article 2: Prelude
- Article 3: Calendar Date Table (pt. 1 & 2)
- Article 4: DateFilter Table
- Article 5: Extracting Column Names from INFO.VIEW.COLUMNS()
- Article 6: Going 88 mph

**Yellow Pages** — Nested Slicer Navigation System
- TBD

**Dealer's Choice** — Multi-Dynamic Measure Selector
- Article 10: Dealer's Choice

**March Madness** — Cascading Rankings
- Articles 11–13

---

## Repository Structure

```
Dim-Mak-Techniques/
├── README.md
├── The Dyson Sphere/
│   ├── README.md
│   └── [Key Design] Dyson Sphere.dax
├── The Delorean/
│   ├── README.md
│   └── [Key Design] The Delorean.dax
├── Dealer's Choice/
│   ├── README.md
│   └── [Key Design] Dealer's Choice.dax
├── March Madness/
│   ├── README.md
│   └── [Key Design] Cascading Ranking.dax
└── Yellow Pages/
    └── README.md
```

---

## Author

**MKienChau**
GitHub: [MKienChau](https://github.com/MKienChau)
