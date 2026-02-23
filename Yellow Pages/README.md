# Yellow Pages: Nested Slicer Navigation System

> A bookmark-driven filter panel architecture that organizes 50+ slicers into a clean, category-based navigation system — without cluttering the report canvas or overwhelming end users.

---

## Design Intent

Most Power BI reports with deep filtering needs end up in one of two places: a cluttered canvas where slicers compete for space, or a separate filter page that breaks the user's flow. Neither is a good experience.

Yellow Pages solves this by treating filters as a navigation system rather than static canvas elements. Slicers are grouped into categories, hidden by default, and revealed on demand through a bookmark-driven panel that opens and closes like a drawer. The user sees a clean report, clicks a category button, a filter panel slides in, they make their selections, and the panel closes. The report updates. No page switching, no visual clutter.

The name reflects the concept: a structured directory you consult when you need something specific, then put away.

---

## What It Looks Like

The system consists of:
- A **navigation bar** with category buttons (Customer, Location, Product, Dates & Time Intelligence, Display Values, etc.)
- **Filter panels** per category — groups of slicers that show/hide as a unit
- **Bookmark pairs** per category controlling show/hide state
- A **close/clear** mechanism to reset the panel state

Each category panel is independent — opening one closes others. The canvas underneath remains fully interactive at all times.

---

## Architecture Pattern

### Layer Structure (Selection Pane)

```
Page
├── Executive Header
├── Matrix / Visuals
├── Page Navigator
├── Calc Slicer
│
├── Group 7 (Filter Panel — e.g. Customer)
│   ├── btn:ShowFilters
│   ├── btn:CloseFilterBox
│   └── BTN:ClearAllFilters
│
├── Group 1 (Navigation Button Bar)
│   ├── Btn:Dates&TimeIntel
│   ├── Btn:DisplayValues
│   ├── Btn:Customer
│   ├── Btn:Location
│   ├── Btn:Products
│   └── Shp:FilterPG (background shape)
│
├── Group 3 (Display Settings Panel)
│   ├── ValueFormat
│   ├── VarDisplay
│   ├── Currency
│   └── DisplaySettings
│
├── Group 4 (Customer Slicers)
│   ├── CustomerIndex
│   ├── AccountType
│   ├── AccountTypeCd
│   └── Customer
│
├── Group 5 (Location Slicers)
├── Group 6 (Product Slicers)
└── ShapeFiltersTog (toggle shape overlay)
```

Groups are the core unit of the architecture. Each category's slicers live inside a Group, which is shown or hidden as a single object via bookmarks.

---

## Bookmark Architecture

### Per Category — 2 Bookmarks Required

| Bookmark | Purpose |
|---|---|
| `Show{Category}` | Sets the category Group to visible, hides all other category Groups |
| `Hide{Category}` | Sets all category Groups to hidden, returns to clean canvas state |

### Navigation Bookmarks

Each navigation button in the button bar triggers a `Show{Category}` bookmark. The close button inside each panel triggers the corresponding `Hide{Category}` bookmark.

### Bookmark Count at Scale

| Categories | Show/Hide Pairs | Navigation Bookmarks | Total |
|---|---|---|---|
| 5 | 10 | 5 | ~15 per page |
| Full implementation (50+ slicers) | 20+ | 10+ | 50+ total |

Bookmark count scales with the number of categories and pages. This is the primary cost of the pattern.

---

## Naming Conventions

Consistent naming is critical for managing this many bookmarks and groups without losing track:

**Groups:** `Group {N}` with Selection Pane labels describing their role (e.g. "Customer Slicers", "Navigation Bar")

**Buttons:** `Btn:{Category}` for navigation, `btn:ShowFilters` / `btn:CloseFilterBox` for panel controls

**Bookmarks:** `{PageName}{Category}` pattern (e.g. `OverviewCustomer`, `OverviewProduct`) — page-scoped to avoid cross-page conflicts

**Shapes:** `Shp:` prefix for background/overlay shapes that are part of the layout but not interactive

---

## Implementation Pattern

1. Build all slicers for a category and group them in the Selection Pane
2. Set the group to hidden by default
3. Create a `Show{Category}` bookmark — capture only the visibility state of all category groups (not filters, not data)
4. Create a `Hide{Category}` bookmark — all groups hidden
5. Create a navigation button, assign the `Show{Category}` bookmark as its action
6. Create a close button inside the panel, assign the `Hide{Category}` bookmark
7. Repeat per category
8. Add a ClearAllFilters button wired to a bookmark that resets all slicer selections

**Critical bookmark setting:** When capturing bookmarks for show/hide, set the bookmark to capture **Current page** and **Selected visuals** only — uncheck Data. This ensures the bookmark controls visibility without resetting filter state.

---

## Known Limitations

- **Not automatable** — Power BI bookmark management has no programmatic API. Tabular Editor, external scripts, and ALM Toolkit cannot create or modify bookmarks. Every bookmark must be created manually in the Power BI Desktop UI.
- **Page duplication breaks bookmarks** — duplicating a report page does not carry over bookmark-to-object bindings. Every bookmark on the duplicated page must be manually re-pointed to the new page's objects.
- **Scale ceiling** — the manual overhead makes this pattern practical up to approximately 5 report pages. A 6th page of the same structure costs disproportionate setup time relative to the UX gain.
- **Bookmark ordering** — the Bookmarks pane has no folder or grouping system. With 50+ bookmarks, navigation within the pane becomes unwieldy. Strict naming conventions are the only mitigation.
- **Group visibility state** — if a visual inside a group is individually toggled outside of the bookmark workflow, it can fall out of sync with the bookmark state and produce unexpected show/hide behavior.

---

## Design Philosophy

The implementation cost is high. The UX return justifies it.

Stakeholders interacting with a Yellow Pages-enabled report experience a clean, navigable interface that meets them where they are — no training required, no explanation of where filters live. The pattern trades developer time for end-user clarity, which is the correct tradeoff in any stakeholder-facing report context.

The tedium of bookmark management is a Power BI platform limitation, not a design flaw. When Microsoft improves bookmark APIs, this pattern becomes significantly easier to implement — the architecture itself is sound.
