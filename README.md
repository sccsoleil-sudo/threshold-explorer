# Auto Credit/Write Off Threshold Explorer

Browser-only tool for exploring how claim / deduction line items behave under different **Amount thresholds**. Two tabs — **COOP** and **Logistics Claims** — load the matching SAP UI5 Excel export, filter by one or more years / months / divisions, and show what share of lines and claim dollars fall at or under a chosen `$X`. Lock scenarios to compare thresholds side by side.

## Purpose

Answer questions like:

- If we treat lines at or under **$400** (COOP) or **$250** (Logistics) as “threshold,” what % of lines and $$ does that cover?
- How does coverage change as the cutoff moves — and where is the elbow (extra $ buys little extra coverage)?
- Does one threshold work across divisions (CPD / PPD / LPD / LDB) or claim types (Shortage / PRX / Returns)?

## Repository layout

```
.
├── threshold-explorer.html          # Entire app (UI + logic)
├── reference-coop-onepager.html     # Static COOP one-pager style reference
├── reference-logistics-onepager.html
└── README.md
```

COOP and Logistics each load their **own** workbook via the tab’s dropzone. Filenames can be anything (e.g. exports with timestamps); only the column schema matters.

## Tech stack

| Layer | Choice |
|--------|--------|
| UI | Single HTML file, vanilla JavaScript |
| Styling | Inline CSS (aligned with one-pager accent colors) |
| Excel parsing | [SheetJS](https://cdn.sheetjs.com) `xlsx` **0.20.3** (CDN) |
| Coverage curves | [Chart.js](https://www.chartjs.org/) **4.4.8** (CDN) |
| Snapshot bars | HTML/CSS bars (one-pager style) |
| Chart download | PNG via canvas export + [html2canvas](https://html2canvas.hertzen.com/) **1.4.1** |
| Runtime | Browser File API + `ArrayBuffer` |

Sensitive AR data stays on the client. Parsed workbooks, filters, and locks are stored in **IndexedDB** in this browser so the next visit restores them without re-uploading. Use **Forget saved … data** to wipe. Network is only needed for CDN scripts unless they are vendored locally.

## How to run

1. Open `threshold-explorer.html` (or `index.html`) in a modern browser.
2. On the **COOP** tab, drop any COOP export; on **Logistics**, drop any logistics claims export (names can differ).
3. Adjust threshold and filters; use **Lock this threshold** to compare options.

## Publish a free website (GitHub Pages)

The repo is ready locally on `main` (Excel files are **not** included — they stay on each user’s machine).

1. Create a **public** repo on GitHub: https://github.com/new  
   Suggested name: `threshold-explorer`
2. In Terminal, from this folder:

```bash
git remote add origin https://github.com/YOUR_USERNAME/threshold-explorer.git
git push -u origin main
```

3. On GitHub: **Settings → Pages → Build and deployment**  
   - Source: **Deploy from a branch**  
   - Branch: **main** / **/ (root)** → Save  
4. After ~1 minute the site is:

`https://YOUR_USERNAME.github.io/threshold-explorer/`

Anyone with the link can open the tool and drop their own Excel files. No claim data is hosted on GitHub.

## Data model

Required columns (fuzzy header match): **Amount**, **Reason Code**. Optional: **Business Area** (Division), **Journal Entry Date** (year/month), **Customer Name** / **Customer** (top-clients table), **Days in Arrears** (priority heat).

Each row becomes:

```js
{ amount, absAmount, reason, baCode, division, divisionLabel, typeLabel, year, month, customer, customerId, daysInArrears }
```

**Under threshold / in band** means either `|Amount| ≤ threshold` or `From ≤ |Amount| ≤ To`, depending on the amount-rule toggle. Dollar totals always use absolute amount.

- **COOP** groups charts by Division (`02AA`→CPD, `02AB`→PPD, `02AC`→LPD, `02AD`→LDB).
- **Logistics** groups by claim type / reason code from the file:
  - **R02** Shortage · **R03** PRX · **R04** Freight/Unloading · **R05** Returns · **R16** FAC · **R17** Short Payment  
    Reason focus is multi-select (empty = all). Live charts also show **by Division for each reason code**.
- Both tabs include a **Top clients** matrix (customer rows × division columns), ranked by total claim $, respecting the current filters and amount band.
- **Priority explorer** tab: choose **Amount only**, **Days only**, or **Both (AND)**. Each factor has Medium / High / Urgent `$` or days starts (Low = below Medium). Heatmap shows Included/Excluded by Minimum level; detail table lists included lines.

## Application logic

```
Load xlsx → parseWorkbook → filteredRows (multi year / month / division [/ type])
  → groupStats(threshold)     → KPIs, one-pager-style bars, detail table
  → computeCoverageCurve()    → coverage vs threshold line charts
  → lockCurrent()             → immutable snapshots for comparison
```

### Coverage vs threshold (CDF sweep)

`computeCoverageCurve` sorts filtered amounts and walks cutoffs from `$0` to the slider max (~80 steps, plus presets). It returns:

1. **Overall curve** — `% lines ≤ thr` and `% $$ ≤ thr` vs threshold  
2. **By group** — `% lines ≤ thr` one series per division (COOP) or type (Logistics)

A dashed vertical marker shows the **current** threshold. Curves are cached per filter set so dragging the slider only moves the marker; recomputation runs when filters or data change.

Use this chart to **choose** `$X`. The paired HTML bars answer “at *this* `$X`, what’s covered by segment?”

### Charts overview

| Chart | Type | Role |
|--------|------|------|
| Coverage vs threshold | Line (CDF) | How coverage grows as $X rises; find the elbow |
| Coverage by division / type | Line | Whether one cutoff fits all segments |
| Live paired bars | HTML (sqrt scale) | Snapshot: threshold $ / lines vs total by segment |
| Priority volume heat | 1D strip or 2D heatmap | Where claim volume sits by amount and/or days |
| Priority mix by reason | Stacked bar | Low→Urgent share per reason code |
| Reason × Priority | Heat table | Volume intensity per reason and priority |
| Detail table | Table | Division × reason (COOP) or type (Logistics) |
| Locked cards | Mini bars | Frozen scenarios for side-by-side compare |

### Core functions

| Function | Role |
|----------|------|
| `parseWorkbook` | Sheet + column map → row model |
| `filteredRows` | Multi-select year / month / division (empty = all); logistics type focus |
| `groupStats` | Under/total aggregates by group at one threshold |
| `computeCoverageCurve` | Threshold sweep for overall + by-group coverage |
| `renderCoverageCharts` | Chart.js CDF + marker (cached) |
| `renderPairedCharts` | One-pager-style HTML bars |
| `lockCurrent` / `renderLocks` | Scenario snapshots |

## Design notes

1. **Zero-install** — one HTML file; sensitive AR data never uploaded.  
2. **Two audiences** — COOP and Logistics share the same engine with different defaults, grouping, and accents.  
3. **CDF for policy choice, bars for communication** — sweep curves for analysis; locked bars for stakeholder compare.  
4. **Excel `$400` parity** — workbook formulas may use positive-only and strict `< 400`; the app uses `|Amount| ≤ thr`.

## Limitations

- No export of charts/tables; locks clear on reload.  
- CDN required unless SheetJS / Chart.js are vendored.  
- Full sheet in memory (fine at ~90k rows).  
- Pivot sheets inside the workbook are not read by the app.
