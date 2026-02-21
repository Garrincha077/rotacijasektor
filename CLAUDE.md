# CLAUDE.md — Rotacija Sektora

This document describes the codebase structure, conventions, and workflows for AI assistants working on this project.

---

## Project Overview

**Rotacija Sektora** is a browser-based market sector rotation tracker for S&P 500 sectors. It allows users to manually enter weekly performance data from finviz.com, then visualize trends, detect rotation signals, and analyze sector consistency over multiple time horizons.

- **Language**: Croatian UI (`lang="hr"`)
- **Architecture**: Single-file SPA — the entire application lives in `index.html`
- **No build step**: Open `index.html` directly in any modern browser
- **No backend**: All data persists in browser `localStorage`

---

## Repository Structure

```
rotacijasektor/
└── index.html      # Entire application (HTML + embedded CSS + embedded JS, ~1392 lines)
```

There are no package managers, build tools, test frameworks, CI/CD pipelines, or additional source files.

---

## Running the Application

```bash
# Option 1 — open directly
open index.html          # macOS
xdg-open index.html      # Linux

# Option 2 — serve locally (avoids potential file:// restrictions)
python3 -m http.server 8080
# then open http://localhost:8080 in a browser
```

No installation, compilation, or environment variables are needed.

---

## File Structure: `index.html`

The file is divided into clearly marked sections using banner comments (`// ═══...`):

| Lines | Section | Description |
|-------|---------|-------------|
| 1–8 | **Head / CDN** | DOCTYPE, charset, viewport, Chart.js 4.4.0 CDN, Google Fonts |
| 9–455 | **`<style>` — CSS** | All styles; dark theme via CSS custom properties |
| 457–680 | **`<body>` — HTML** | 5 tab panels + navigation bar + toast notification element |
| 681–700 | **KONSTANTE** | Global constants (`SEKTORI`, `PERIODI`, `STORAGE_KEY`) |
| 701–704 | **Global state** | `sortRankBy`, `sortDir`, `vizPeriod`, `charts` |
| 706–715 | **STORAGE** | `getData()` / `saveData()` wrapping localStorage |
| 717–775 | **INIT** | `init()`, `updateClock()`, `buildSectorGrid()`, `buildSectorSelects()`, `sanitize()` |
| 777–870 | **SAVE / CLEAR** | `saveEntry()`, `clearForm()`, `updateStats()`, `populateWeekSelect()` |
| 872–931 | **RANG TABLICA** | `renderRankTable()`, `rankSort()` |
| 934–957 | **HEATMAP** | `renderHeatmap()`, `hmColor()` |
| 959–1120 | **CHARTS** | `renderCharts()`, `renderRadar()`, `chartOptions()`, `barOptions()` |
| 1122–1218 | **SIGNALS** | `renderSignals()`, `renderConsistency()` |
| 1220–1338 | **HISTORY** | `renderHistory()`, `loadEntry()`, `deleteEntry()`, `deleteAll()` |
| 1277–1338 | **EXPORT / IMPORT** | `exportJSON()`, `exportCSV()`, `importJSON()`, `handleImport()` |
| 1340–1367 | **HELPERS** | `pct()`, `valCell()`, `avg()`, `showToast()` |
| 1369–1390 | **TAB NAVIGATION** | `showTab()` + `init()` boot call |

---

## Core Data Model

### Constants

```javascript
const SEKTORI = [
  'Basic Materials', 'Communication Services', 'Consumer Cyclical',
  'Consumer Defensive', 'Energy', 'Financial',
  'Healthcare', 'Industrials', 'Real Estate', 'Technology', 'Utilities'
]; // 11 sectors, fixed order

const PERIODI = [
  { key: 'tjedno',   label: 'Tjedno'  },  // Weekly
  { key: 'miesecno', label: 'Mjes.'   },  // Monthly
  { key: 'tromj',    label: '3 Mjes.' },  // 3-Month / Quarterly
  { key: 'polugod',  label: '6 Mjes.' },  // 6-Month / Semi-annual
  { key: 'godisnje', label: 'God.'    },  // Annual / 1-Year
  { key: 'ytd',      label: 'YTD'     },  // Year-to-date
]; // 6 time periods, fixed order

const STORAGE_KEY = 'sektorRotacija_v2'; // localStorage key
```

### Entry Object (stored in `localStorage`)

```javascript
{
  id: 1706745600000,        // Unix timestamp (ms), used as unique ID
  datum: '2024-02-01',      // ISO date string (YYYY-MM-DD)
  label: '2024-02-01',      // Optional human label (defaults to datum)
  sektori: {
    'Basic Materials': {
      tjedno:   1.23,        // float, percentage return
      miesecno: -0.45,
      tromj:    3.10,
      polugod:  5.67,
      godisnje: 8.90,
      ytd:      2.34
    },
    'Communication Services': { /* same shape */ },
    // ... all 11 sectors
  }
}
```

Data array is stored sorted ascending by `datum`. The localStorage value is `JSON.stringify(entries[])`.

---

## Tab Structure (UI)

| Tab ID | Croatian | Purpose |
|--------|----------|---------|
| `unos` | Unos | Data entry — sector grid form + quick stats |
| `rang` | Rang Tablica | Ranking table + heatmap; sortable by any period |
| `vizual` | Vizualizacije | Line chart, bar chart, radar chart |
| `signali` | Signali | Rotation signal detection + consistency table |
| `povijest` | Povijest | History list + edit/delete/import/export |

Tab switching is handled by `showTab(tabName)`. Each tab has a corresponding render function called on activation.

---

## Key Functions

### Data I/O
| Function | Description |
|----------|-------------|
| `getData()` | Reads and parses localStorage; returns `[]` on error |
| `saveData(arr)` | Serializes array to localStorage |
| `saveEntry()` | Reads all inputs, builds entry object, upserts by date |
| `loadEntry(id)` | Populates form inputs from a stored entry (for editing) |
| `deleteEntry(id)` | Removes an entry by id after confirm dialog |
| `importJSON()` / `handleImport()` | Merge-imports from a JSON file; deduplicates by `id` or `datum` |
| `exportJSON()` / `exportCSV()` | Triggers browser download of all data |

### Rendering
| Function | Description |
|----------|-------------|
| `renderRankTable()` | Sorts and renders the ranking table for the selected week |
| `renderHeatmap(entry)` | Renders color-coded performance grid for an entry |
| `renderCharts()` | Destroys existing Chart.js instances and redraws all three charts |
| `renderRadar()` | Redraws radar chart for a selected sector |
| `renderSignals()` | Applies signal thresholds to latest entry, renders matches |
| `renderConsistency(data)` | 4-week rolling average ranks + trend arrow per sector |
| `renderHistory()` | Lists all entries newest-first |

### Utilities
| Function | Signature | Description |
|----------|-----------|-------------|
| `pct(v)` | `number → string` | Formats as `+1.23%` or `–` for null |
| `valCell(v)` | `number → string` | Returns color-coded `<span>` HTML |
| `avg(arr)` | `number[] → number` | Mean of non-null values |
| `sanitize(s)` | `string → string` | Strips non-alpha chars for DOM IDs |
| `showToast(msg, isError)` | — | Shows a 3-second notification |
| `hmColor(v)` | `number → string` | Inline CSS string for heatmap cell |

---

## Signal Detection Logic

The "Signali" tab identifies potential **dead cat bounces** or **bear trap** rotations. A signal fires when all three conditions are met using the most recent entry:

```
weekly  >= sigTjedno  (default: +1.5%)
monthly >= sigMjesecno (default: +0.0%)
AND (annual <= sigGodisnje OR semi-annual <= sigGodisnje)  (default: -5%)
```

The thresholds are user-configurable via inputs on the Signali tab. The signal logic is in `renderSignals()` at line 1125.

---

## Chart System

All charts use **Chart.js 4.4.0** loaded from CDN. The `charts` global object holds references:

- `charts.trend` — Line chart: sector performance over time for the selected period
- `charts.bar` — Bar chart: latest week's cross-sector performance comparison
- `charts.radar` — Radar chart: one sector across all 6 time periods

Charts are destroyed and rebuilt on each `renderCharts()` call. Failure to destroy before recreating causes Canvas errors. The pattern is:

```javascript
Object.values(charts).forEach(c => c && c.destroy());
charts = {};
```

---

## CSS / Theme System

All colors are defined as CSS custom properties on `:root` (lines 10–28):

```css
:root {
  --bg, --bg2, --bg3, --bg4   /* background layers */
  --border                     /* border color */
  --amber, --amber-dim         /* accent / warning */
  --green, --green-dim         /* positive / gain */
  --red, --red-dim             /* negative / loss */
  --blue                       /* info accent */
  --text, --text-dim, --text-bright  /* typography */
  --signal, --signal-bg        /* rotation signal highlight */
}
```

Typography: `JetBrains Mono` for data/numbers, `Rajdhani` for headings/labels.

Value coloring classes (used in `valCell()`):
- `.val-pos2` — gain > 3%
- `.val-pos1` — gain 0–3%
- `.val-neg1` — loss 0–3%
- `.val-neg2` — loss > 3%
- `.val-zero` — null/missing

---

## Conventions for Making Changes

### Adding a new sector
1. Add its name to `SEKTORI` (line 685). The grid, selects, charts, and tables are all built from this array dynamically.

### Adding a new time period
1. Add `{ key: '...', label: '...' }` to `PERIODI` (line 691).
2. Add the corresponding column header to the HTML table in the "Rang" tab body (`<thead>` in tab-rang).
3. Add the corresponding sort button if needed.

### Modifying signal thresholds or logic
- Edit `renderSignals()` (line 1125). The condition is at line 1144.
- Default values are set in the HTML `<input>` elements inside `tab-signali`.

### Adding a new chart
- Create a new canvas element in `tab-vizual`.
- Add a new key to `charts` and follow the destroy-before-render pattern.

### Changing storage structure
- Bump `STORAGE_KEY` to `sektorRotacija_v3` (or next version) to avoid conflicts with existing data.
- Add migration logic in `getData()` if backward compatibility is needed.

### Inline styles vs CSS classes
The codebase mixes both. Prefer CSS classes for reusable styles; inline styles are used for dynamically computed values (colors from `hmColor()`, trend colors in `renderConsistency()`).

---

## Data Source

Performance data is manually entered from **[finviz.com/groups.ashx](https://finviz.com/groups.ashx)**. Users select "Sector" grouping, then record the performance percentages for each time horizon. The app does not fetch data automatically.

---

## Localization Notes

- UI language: **Croatian**
- Date display: Croatian locale (`hr-HR`) via `toLocaleDateString`/`toLocaleTimeString`
- Key Croatian terms used in code:
  - `datum` = date
  - `tjedno` = weekly
  - `miesecno` = monthly
  - `tromj` = quarterly (3 months)
  - `polugod` = semi-annual (6 months)
  - `godisnje` = annual
  - `sektor` = sector
  - `unos` = entry/input
  - `povijest` = history
  - `rang` = ranking
  - `signali` = signals

---

## No Tests, No Build

This project has no automated tests and no build pipeline. Validate changes by:
1. Opening `index.html` in a browser
2. Entering sample data in the "Unos" tab
3. Navigating all five tabs to verify rendering
4. Testing export/import round-trip via the "Povijest" tab
5. Checking browser console for JavaScript errors
