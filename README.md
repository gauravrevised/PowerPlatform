# LearnHub Visual — PCF Component (Power BI-style)

One control, eight visual types. Drop an instance on the canvas per
chart/KPI you need — exactly like adding a new visual in Power BI —
pick its **Visual Type** from the property dropdown, then configure
its data and styling independently. To rebuild the full analytics
dashboard, place ~10-14 instances of this control in a Container grid
on one screen (see "Composing the dashboard" below).

## Why this replaces the earlier single fixed-layout control

The first version baked the whole dashboard's layout into one
component with one big JSON blob. That doesn't match how Power BI
(or any real BI tool) actually works — every visual is independent:
its own data, its own colors, its own fonts, addable/removable/
resizable without touching anything else. This version mirrors that.

## Properties (the "format pane")

| Property | Type | Purpose |
|---|---|---|
| `visualType` | Enum | KPI Card / Line Chart / Bar Chart / Stacked Bar / Donut / Gauge / Funnel / Table |
| `chartData` | JSON text | Data — shape depends on `visualType`, see below |
| `showTitle` | Bool | Show/hide title bar |
| `chartTitle` | Text | Title text |
| `titleFontSize` / `titleFontColor` / `titleFontWeight` | Number / Hex / Enum | Title styling |
| `titleAlign` | Enum | Left / Center / Right — horizontal position of the title within the header |
| `showSubtitle` / `subtitleText` | Bool / Text | Small metadata label pinned to the right of the header regardless of title alignment — e.g. "Daily · Last 30 days". Styled using the Label font settings. |
| `kpiIcon` | Enum | 20 built-in icons + "None" — only applies when `visualType` = KPI Card |
| `valueFontSize` / `valueFontColor` | Number / Hex | Styling for the KPI big number, gauge %, table values |
| `labelFontSize` / `labelFontColor` | Number / Hex | Styling for axis labels, legend, table headers |
| `fontFamily` | Text | CSS font stack for the whole visual |
| `showLegend` | Bool | Show/hide legend (Line/Bar/Stacked/Donut only) |
| `accentColor` | Hex | Primary series/bar/KPI color |
| `secondaryColor` | Hex | Secondary series color (2nd line, "fail" bar, etc.) |
| `backgroundColor` / `borderColor` | Hex | Card background/border — blank = theme default |
| `cornerRadius` | Number | Card corner radius (px) |
| `themeMode` | Enum | Dark/Light — supplies defaults for anything left blank above |

Any color left blank falls back to the theme default (dark or light)
rather than breaking — same behavior as Power BI's "Auto" color state.

## Header layout — title, alignment, and subtitle

The header row (title + subtitle) sits above every visual type, not
just KPI. `titleAlign` controls where the title text sits (left,
center, or right) by giving it `flex:1` with `text-align` set
accordingly; the subtitle is a separate `flex:none` element that
always lands at the true right edge of the header regardless of the
title's own alignment, since it comes immediately after the title's
flexible box. So `titleAlign: Right` + a subtitle will show the title
pushed toward — but not colliding with — the subtitle; `titleAlign:
Left` (the default) puts them at opposite ends, matching the original
Grafana-style mockup ("Visits vs Unique Visitors" ... "Daily · last
30 days").

## KPI icon

Set via the `kpiIcon` dropdown property (not inside `chartData`) —
20 built-in icons covering general + retail/CPG use cases (Visits,
Users, Assessment, Check, Star, Chat, Trophy, Learning Path, Cart,
Price Tag, Barcode, Truck, Warehouse, Package, Receipt, Storefront,
Trending Up, Calendar, Percent, Target), plus "None" to hide it. The
icon renders top-right of the KPI card, tinted with `accentColor`.
Only rendered when `visualType = KPI Card` — the property is ignored
for other visual types.

Per-series/per-segment colors set **inside** `chartData` (e.g. each
series' `"color"` field) always win over `accentColor`/`secondaryColor`
— those two properties are just the fallback when the data doesn't
specify its own colors.

## `chartData` schema per Visual Type

### KPI Card
```json
{ "value": "18,420", "trend": 12.4, "trendDirection": "up", "sparkline": [24,20,22,14,16,10,12,6] }
```

### Line Chart
```json
{
  "categories": ["Mon","Tue","Wed","Thu","Fri"],
  "series": [
    { "name": "Visits", "color": "#5794F2", "data": [420,460,440,510,490] },
    { "name": "Unique Visitors", "color": "#33D6C0", "data": [180,190,185,210,205] }
  ]
}
```
(`categories` is accepted for future axis-label rendering; current
version scales purely off array length/values — safe to omit.)

### Bar Chart (grouped) / Stacked Bar Chart
Same shape as Line Chart. `visualType = 2` renders grouped bars side
by side per category; `visualType = 3` stacks them.
```json
{
  "categories": ["W1","W2","W3","W4"],
  "series": [
    { "name": "Passed", "color": "#5BC98A", "data": [68,74,71,79] },
    { "name": "Failed", "color": "#F2495C", "data": [32,26,29,21] }
  ]
}
```

### Donut Chart
```json
{
  "segments": [
    { "label": "Destination", "value": 45, "color": "#5794F2" },
    { "label": "Routine", "value": 35, "color": "#33D6C0" },
    { "label": "Seasonal", "value": 20, "color": "#FFB300" }
  ]
}
```

### Gauge
```json
{ "value": 97 }
```

### Funnel
```json
{
  "steps": [
    { "name": "Assessments taken", "value": 1096, "pct": 100, "color": "#5794F2" },
    { "name": "Recommendations generated", "value": 1062, "pct": 97, "color": "#33D6C0" },
    { "name": "Course started", "value": 512, "pct": 47, "color": "#FFB300" }
  ]
}
```

### Table
`columns[].type` controls how each cell renders: `text` (default),
`number`, `bar` (renders a mini progress bar — value should be 0-100),
or `badge` (renders a colored pill — green if ≥50, amber if <50).
```json
{
  "columns": [
    { "key": "name", "label": "Course" },
    { "key": "recommended", "label": "Recommended", "type": "number" },
    { "key": "startRate", "label": "Start Rate", "type": "bar" },
    { "key": "completion", "label": "Completion", "type": "badge" }
  ],
  "rows": [
    { "name": "Incremental Lift Modeling", "recommended": 214, "startRate": 58, "completion": 62 },
    { "name": "Post-Promo ROI Deep Dive", "recommended": 198, "startRate": 51, "completion": 57 }
  ]
}
```

## Composing the dashboard from multiple instances

On your Analytics screen, lay out a **Container** grid (matching the
original mockup's structure) and drop one `Visual` control into each
cell:

```
Row 1 (6 columns, KPI Card instances):
  Total Visits | Unique Visitors | Assessments Taken | Pass Rate | Courses Recommended | Chatbot Conversations

Row 2 (2 columns):
  Line Chart (Visits vs Unique Visitors)  |  Stacked Bar (Pass/Fail)

Row 3 (2 columns):
  Funnel (Recommendation funnel)  |  Table (Top recommended courses)

Row 4 (2 columns):
  Gauge (Coverage)  |  Gauge (Accepted)
```

Each instance's `chartData` property gets its own Power Fx formula —
typically a `JSON()` call over whatever slice of your data source that
visual needs. Since each is independent, an app maker can resize,
reorder, remove, or re-theme any single visual without affecting the
others — same flexibility as rearranging a Power BI report canvas.

## Setup (run locally — requires network + Node.js + Power Platform CLI)

This sandbox has no network access, so these files were written by
hand rather than scaffolded/compiled — treat this as a strong first
draft to compile and fix up locally, not guaranteed-correct code.

```bash
pac install latest
pac pcf init --namespace LearnHub --name Visual --template field
# ^ copy this folder's ControlManifest.Input.xml / index.ts / css
#   over the generated stub files, keeping the auto-generated
#   generated/ManifestTypes.d.ts that pac creates

npm install
npm start          # local test harness preview
npm run build
pac pcf push --publisher-prefix <yourprefix>
```

## Known gaps to check after compiling

- Table doesn't virtualize/paginate — many rows will just render tall
- Line/Bar charts don't render category axis labels yet (data-driven
  geometry only) — worth adding if precise x-axis reading matters
- No built-in error boundary beyond the top-level JSON.parse try/catch
- `titleFontWeight` enum maps to fixed CSS weights (400/600/800) —
  adjust `FONT_WEIGHTS` in index.ts if your font doesn't support those
