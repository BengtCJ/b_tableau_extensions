# Brand Intelligence Dashboard — Chart Library Implementation

> Read this entire file before writing any code. Implement charts in the order listed.
> Every hex value, D3 pattern, and data guard is mandatory. Do not substitute or simplify.

---

## Metric → Chart quick reference

| Indicator ID | Metric name | **Recommended chart** | Alternatives |
|---|---|---|---|
| `tam` | Total Addressable Market | **BAN** | Treemap (if SAM/SOM added) |
| `cagr` | Compound Annual Growth Rate | **BAN** | — |
| `svt` | Search Volume & Trends | **Straight line** | Smooth line |
| `sop` | Share of Preference | **Scale · Figma** | Slope chart |
| `cra` | Conversion Rate Analysis | **Horizontal bar** | — |
| `bss` | Branded Search Share | **Stacked 100% area** (trend) · **Treemap bar** (snapshot) | Stream, Slope, Small multiples |
| `dvtr` | Distinctive Verbal Tone Recognition | **BANs** | Horizontal bar, Scale · Figma |
| `sstsr` | Scroll-stop / Thumb-stop Ratio | **Horizontal bar** | — |
| `eqr` | Engagement Quality Ratio | **Vertical bar stacked** | Inset bubble, Progress ring, Waffle |
| `sov` | Share of Voice | **Stacked 100% area** (trend) · **Treemap bar** (snapshot) | Bubbles, Inset bubble, Stream |
| `vom` | Velocity of Mentions | **Smooth line** | Straight line |
| `nps` | Brand Loyalty / NPS | **Waffle chart** | Dot matrix, Inset bubble, Arc (Illy callout) |
| `ba` | Brand Affinity | **Multi-scale dots** | — |
| `ebl` | Brand Engagement beyond likes | **Vertical bar stacked** | Inset bubble |
| `bt` | Brand Trust | **Scale · Figma** | BANs |

---

## Project setup

```bash
npm init -y
npm install d3@7
```

```
src/
  tokens.css
  lib/
    scaffold.js     # base SVG + ResizeObserver pattern
    selection.js    # brand selection event bus
    tooltip.js      # shared tooltip component
    data-guards.js  # validate data before rendering
  charts/
    ban.js          # tam, cagr
    bans.js         # dvtr, bt (alt), nps (alt)
    line.js         # svt (straight), vom (smooth)
    slope.js        # bss, sov, sop (alt)
    area-100.js     # bss, sov
    stream.js       # bss (alt), sov (alt)
    treemap.js      # bss (snapshot), sov (snapshot)
    treemap-bar.js  # bss, sov — single wide bar
    bubbles.js      # sov (alt), bss (alt)
    inset-bubble.js # sov, eqr, ebl, nps (alt)
    progress-ring.js# eqr (alt), nps (alt)
    waffle.js       # nps, eqr (alt)
    hbar.js         # cra, sstsr, dvtr (alt)
    vbar-stacked.js # eqr, ebl
    scale-figma.js  # sop, bt, dvtr (alt)
    multiscale.js   # ba
    dot-matrix.js   # nps (alt)
    small-multiples.js # bss (context), sov (context)
    arc.js          # nps (Illy callout — HOLD)
  instances/
    tam.js cagr.js svt.js sop.js cra.js bss.js
    dvtr.js sstsr.js eqr.js sov.js vom.js nps.js
    ba.js ebl.js bt.js
```

---

## Design tokens

Create `src/tokens.css` — all charts import these. Never hardcode hex values in chart files.

```css
:root {
  /* Greys — linear scale, 900 = darkest */
  --grey-900:  #1A1A1A;   /* page background */
  --grey-800:  #333333;   /* card surfaces, tooltip bg */
  --grey-700:  #4D4D4D;   /* passive fills, empty waffle dots */
  --grey-500:  #808080;   /* competitor fills, lines */
  --grey-300:  #B3B3B3;   /* axis labels, secondary text */
  --grey-100:  #E6E6E6;   /* primary text, active labels */

  /* Primary — Illy brand only. DO NOT use primary-1000 anywhere. */
  --primary-500:  #E994A2;  /* main Illy fill — bars, lines, arcs, circles */
  --primary-700:  #E0697D;  /* passive segments for Illy (stacked bar bottom) */
  --primary-900:  #D73F58;  /* negative semantic — detractors, decline */

  /* Semantic */
  --positive:   #E994A2;   /* re-use primary-500 */
  --negative:   #D73F58;   /* re-use primary-900 */
  --neutral:    #4D4D4D;   /* passives, grey-700 */
  --benchmark:  #B3B3B3;   /* reference lines, grey-300 */
}
```

---

## Typography

```
Baskerville Italic  →  all large display numbers (BAN, BANs, value inside arcs/rings)
Tableau Light       →  axis labels, brand name labels, secondary text (9–11px)
Tableau Regular     →  chart titles, tooltip body (10–12px)
```

Font size rules:
- BAN headline: `80px` Baskerville italic, `color: var(--primary-500)`
- BANs — Illy value: `56px` · Competitor values: `32px`
- Direct data labels: `11–13px` Baskerville italic, brand colour
- Axis tick labels: `9–10px` Tableau Light, `var(--grey-500)`
- Brand name labels: `9–10px` Tableau Light uppercase, `var(--primary-500)` (Illy) or `var(--grey-100)` (competitors)
- Benchmark / avg labels: `8.5–9px` Tableau Light uppercase, `var(--grey-700)`

---

## Tableau layout constraints

```js
// Every chart module must use these constants
const MARGIN = { top: 20, right: 110, bottom: 36, left: 52 };
const MARGIN_COMPACT = { top: 8, right: 80, bottom: 20, left: 36 };
const MIN_HEIGHT = 120;   // render "too small" message below this
const isCompact = h => h < 220;
const yTickCount = h => h > 300 ? 5 : h > 150 ? 3 : 2;

// Truncate brand labels at narrow widths
const truncateLabel = (name, containerWidth) =>
  containerWidth < 500 ? name.slice(0, 4) + '…' : name.toUpperCase();
```

SVG must always use `width: 100%; height: 100%; display: block` and read actual dimensions
via `getBoundingClientRect()` at render time. Never set fixed pixel dimensions on SVG.

Debounce ResizeObserver re-renders by `150ms`. No transition on resize — cancel any
in-progress transitions before re-render.

---

## Shared systems

### src/lib/scaffold.js

```js
export function buildChart(containerId, drawFn) {
  const container = document.getElementById(containerId);
  const svg = d3.select(container).append('svg')
    .style('width', '100%').style('height', '100%').style('display', 'block');
  const g = svg.append('g');

  function getDims() {
    const { width, height } = container.getBoundingClientRect();
    const M = height < 220
      ? { top:8, right:80, bottom:20, left:36 }
      : { top:20, right:110, bottom:36, left:52 };
    return { width, height, M,
      iW: width  - M.left - M.right,
      iH: height - M.top  - M.bottom };
  }

  function render() {
    const dims = getDims();
    if (dims.height < 120) { renderTooSmall(svg, dims); return; }
    g.attr('transform', `translate(${dims.M.left},${dims.M.top})`);
    g.selectAll('*').remove();
    drawFn(g, svg, dims);
  }

  const ro = new ResizeObserver(() => {
    clearTimeout(render._t);
    render._t = setTimeout(render, 150);
  });
  ro.observe(container);

  return { render, svg, g };
}

function renderTooSmall(svg, { width, height }) {
  svg.selectAll('*').remove();
  svg.append('text').attr('x', width / 2).attr('y', height / 2)
    .attr('text-anchor', 'middle').attr('fill', 'var(--grey-700)')
    .attr('font-family', 'Arial, sans-serif').attr('font-size', 11)
    .text('Container too small');
}
```

### src/lib/selection.js

```js
let _selected = null;
const _listeners = new Set();

export function broadcastSelection(brand) {
  _selected = _selected === brand ? null : brand;
  _listeners.forEach(fn => fn(_selected));
  // Tableau filter
  try {
    const ws = tableau.extensions.dashboardContent.dashboard.worksheets
      .find(w => w.name === window.__CHART_CONFIG__?.sourceWorksheet);
    if (ws) {
      _selected
        ? ws.applyFilterAsync('Brand', [_selected], FilterUpdateType.Replace)
        : ws.clearFilterAsync('Brand');
    }
  } catch (e) { /* graceful degrade — selection still applied visually */ }
}

export function onSelectionChange(fn) {
  _listeners.add(fn);
  return () => _listeners.delete(fn);
}

// Apply opacity transitions to all [data-brand] elements in a D3 selection
export function applySelectionOpacity(gSelection, selected) {
  gSelection.selectAll('[data-brand]')
    .transition().duration(180).ease(d3.easeCubicOut)
    .attr('opacity', function () {
      const b = this.getAttribute('data-brand');
      if (!selected) return 1;
      if (b === selected) return 1;
      if (b === 'Illy') return 0.35;   // Illy never dims below 0.35
      return 0.2;
    });
}

export const getSelected = () => _selected;
```

**Z-order rule — mandatory for ALL charts:**
Before binding data to marks, sort the array so Illy is always last:
```js
const sorted = [
  ...data.filter(d => d.brand !== 'Illy'),
  ...data.filter(d => d.brand === 'Illy'),  // renders on top in SVG
];
```

### src/lib/tooltip.js

```js
export const tooltip = {
  el: null,
  init() {
    this.el = document.createElement('div');
    Object.assign(this.el.style, {
      position: 'fixed', background: 'var(--grey-800)',
      border: '1px solid var(--grey-700)', borderRadius: '6px',
      padding: '10px 14px', pointerEvents: 'none',
      zIndex: '9999', maxWidth: '200px', display: 'none',
      fontFamily: 'Arial, sans-serif',
    });
    document.body.appendChild(this.el);
  },
  show(event, headerText, rows) {
    // rows: [{ brand, label, value }]
    const hdr = `<div style="font-size:11px;color:var(--grey-500);
      letter-spacing:.06em;text-transform:uppercase;
      border-bottom:1px solid var(--grey-700);padding-bottom:6px;
      margin-bottom:6px">${headerText}</div>`;
    const body = rows.map(r => `
      <div style="display:flex;justify-content:space-between;gap:16px;padding:2px 0">
        <span style="display:flex;align-items:center;gap:6px;font-size:11px;color:var(--grey-300)">
          <span style="width:6px;height:6px;border-radius:50%;
            background:${r.brand==='Illy'?'var(--primary-500)':'var(--grey-500)'}"></span>
          ${r.label}
        </span>
        <span style="font-size:11px;color:${r.brand==='Illy'
          ?'var(--primary-500)':'var(--grey-100)'}">${r.value}</span>
      </div>`).join('');
    this.el.innerHTML = hdr + body;
    this.el.style.display = 'block';
    this._pos(event);
  },
  _pos(event) {
    const { innerWidth: W, innerHeight: H } = window;
    const { width: w, height: h } = this.el.getBoundingClientRect();
    const x = event.clientX + 12, y = event.clientY - h / 2;
    this.el.style.left = (x + w > W ? x - w - 24 : x) + 'px';
    this.el.style.top  = Math.max(8, Math.min(H - h - 8, y)) + 'px';
  },
  hide() { this.el.style.display = 'none'; },
};
```

### src/lib/data-guards.js

Run these before rendering any chart. Log warnings but do NOT throw errors — degrade
gracefully by showing an empty state.

```js
export function guardLikertNotPercent(data, metricId) {
  if (data.some(d => d.value > 1 && d.value <= 5)) {
    console.warn(`[${metricId}] Data appears to be Likert means (1–5 scale), not %.
      Expected percentage values 0–100. Skipping render.`);
    return false;
  }
  return true;
}

export function guardStarbucksPresent(data, metricId) {
  if (!data.some(d => d.brand === 'Starbucks')) {
    console.warn(`[${metricId}] Starbucks missing from data.`);
  }
}

export function guardSvtZeros(data) {
  const zeroMonths = data.filter(d =>
    ['Sep','Oct','Nov','Dec'].includes(d.month) && d.value === 0
  );
  if (zeroMonths.length > 0)
    console.warn(`[svt] Q4 values are all zero — pipeline likely unpopulated.
      Render anyway but flag in UI.`);
}

export function guardNpsRecalculated(data) {
  if (data.some(d => d.nps > 5)) return true; // looks like real NPS
  console.warn(`[nps] Values appear to be Likert means, not NPS scores (-100 to +100).
    Arc chart blocked. Use waffle/BANs until recalculated.`);
  return false;
}

export function guardBaAttributes(data) {
  if (!data[0]?.Trust) {
    console.warn(`[ba] Affinity attribute columns not found.
      Multi-scale requires Trust, Innovation, Value, Style, Heritage columns.`);
    return false;
  }
  return true;
}
```

---

## Chart implementations

### 1 · BAN
**Metrics:** `tam` · `cagr`

Pure HTML — no SVG or D3 needed.

```
Layout: card container (background: var(--grey-800), border-radius: 8px,
        border: 1px solid var(--grey-700), padding: 20px 24px)
Label:  10px Tableau Light, uppercase, var(--primary-500), letter-spacing 0.1em
Value:  80px Baskerville italic, var(--primary-500), line-height 0.9
Unit:   shown inline with value (e.g. "$14.4B", "1.9%")
Period: 9px Tableau Light, var(--grey-700), bottom-right absolute
```

For `cagr`: prefix value with `▲` (green tint) or `▼` (primary-900) based on sign.
No click interaction — display only. Loading state: shimmer rect
`background: linear-gradient(90deg, var(--grey-800) 25%, var(--grey-700) 50%, var(--grey-800) 75%)`
animated 1.5s linear infinite. Empty state: show `—` in var(--grey-700).

Data shape:
```js
{ value: 14.425, unit: 'B', prefix: '$', period: '2025', label: 'TOTAL ADDRESSABLE MARKET' }
```

---

### 2 · BANs
**Metrics:** `dvtr` (recommended) · `bt` (alternative) · `nps` (alternative, only if NPS recalculation fails)

```
Layout: flex row, align-items: flex-end
Illy block:   border-right: 1px solid var(--grey-700), padding-right: 24px, margin-right: 24px
  Label:  7.5px Tableau Light, uppercase, var(--primary-500)
  Value:  56px Baskerville italic, var(--primary-500), line-height 0.92
Competitor blocks (sorted desc by value):
  Label:  7px Tableau Light, uppercase, var(--grey-500)
  Value:  32px Baskerville italic, var(--grey-500)
```

Apply `data-brand` attribute to each value element for selection opacity.
No axes. No chart. The typography IS the chart.

Data shape:
```js
[{ brand: 'Illy', value: 3.04 }, { brand: 'Peets', value: 3.34 }, ...]
```

---

### 3 · Straight line
**Metrics:** `svt` (recommended) · `vom` (alternative)

```
Curve:       d3.curveLinear — angular, connects data points with straight segments
Illy line:   stroke: var(--primary-500), stroke-width: 2
Comp lines:  stroke: var(--grey-500), stroke-width: 1
Zero rule:   1px var(--grey-500) at opacity 0.55 when y=0 is within chart bounds
End dots:    circle r=3 (Illy) r=2 (competitors) at first and last data point
End labels:  right margin, 9px Tableau Light, brand colour
Axes:        x: scalePoint, d3.axisBottom, no domain line. y: scaleLinear,
             tickSize=-iW with grey-700 dashed gridlines, +.0% format
```

Use `guardSvtZeros(data)` before rendering `svt`. If Q4 zeros detected, render with
a `⚠ Q4 data pending` annotation at 9px Tableau Light var(--grey-700).

Data shape:
```js
[{ month: 'Feb', Starbucks: 0.189, Nespresso: -0.017, Peets: 0.07, Lavazza: 0.08, Illy: 0.025 }, ...]
```

---

### 4 · Smooth line
**Metrics:** `vom` (recommended) · `svt` (alternative)

Identical to Straight line except:
```
Curve: d3.curveMonotoneX — smooth interpolation, good for trend direction
```

All other spec identical to Straight line. Implement as `config.curve` parameter
so both line types share the same module:
```js
// In line.js
const curve = config.straight ? d3.curveLinear : d3.curveMonotoneX;
```

---

### 5 · Slope chart
**Metrics:** `bss` (recommended) · `sov` (alternative) · `sop` (alternative, two survey periods)

```
Two x-axis poles: left = period 1, right = period 2
  Vertical lines at x=0 and x=iW, stroke: var(--grey-700), stroke-width: 0.5
  Period labels above poles: 9px Tableau Light, uppercase, var(--grey-300)
Lines:
  Illy: stroke: var(--primary-500), stroke-width: 2.5
  Competitors: stroke: var(--grey-500), stroke-width: 1
End dots: r=5 (Illy), r=4 (competitors), filled brand colour
Labels left of left pole: "BRAN 0.0%" — brand (4-char) + value
Labels right of right pole: same format
Collision detection: if two labels within 11px vertically, offset the lower
  by +11px. Apply to each side independently.
Y scale: d3.scaleLinear, domain [0, max*1.1], no y-axis rendered
No gridlines, no x-axis ticks
```

Data shape:
```js
[{ brand: 'Illy', v1: 0.57, v2: 0.59 }, { brand: 'Starbucks', v1: 82, v2: 83.9 }, ...]
// v1 = start period, v2 = end period, values in same unit as metric
```

---

### 6 · Stacked 100% area
**Metrics:** `bss` (trend view) · `sov` (trend view)

```
Stack: d3.stack().keys(brandOrder).offset(d3.stackOffsetExpand)
Brand order: dominant brand (largest avg share) at bottom array index → Illy last (top)
Curve: d3.curveMonotoneX
Fills — Illy: var(--primary-500) at fill-opacity 0.78
         Competitors: var(--grey-500) at fill-opacity 0.78, 0.62, 0.48, 0.34 by share rank
Boundaries: stroke: var(--grey-900), stroke-width: 0.5 between each layer
Y-axis: 0–100%, d3.format('.0%'), 3 ticks
X-axis: scalePoint or scaleTime, month labels
Inline labels at rightmost point (right margin):
  brand 4-char, 9px Tableau Light, brand colour
  collision offset: ±8px if within 12px vertically
Time brush (optional, enabled via config.brush):
  d3.brushX, 28px strip below x-axis
  handle colour: var(--primary-500)
  double-click resets to full range
```

Click a fill layer → isolation mode: selected layer at 0.78, all others 0.12,
Illy minimum 0.35. Click background → restore all.

Data shape:
```js
[{ month: 'Jan', Starbucks: 82, Nespresso: 10.7, Peets: 6, Lavazza: 0.72, Illy: 0.57 }, ...]
```

---

### 7 · Stream graph
**Metrics:** `bss` (alternative) · `sov` (alternative)

```
Stack: d3.stack().keys(brandOrder).offset(d3.stackOffsetSilhouette)
Curve: d3.curveBasis — organic, flowing
Fills: same opacity rules as stacked area
No y-axis — silhouette offset makes y meaningless
X-axis only (months), 9px Tableau Light
Mid-point labels: text centred in each band at chart midpoint, fill: var(--grey-900)
```

Note in comments: "Stream offset distorts absolute values. Use for visual impression,
not precision reading. Recommend pairing with treemap bar for exact values."

---

### 8 · Treemap
**Metrics:** `bss` (snapshot) · `sov` (snapshot) · `tam` (if SAM/SOM data added)

```
d3.treemap().size([width, height]).padding(2).round(true)
Data: latest single period only — not time series
Sort: d3.treemap sorts by value desc automatically
Fills: Illy var(--primary-500) at opacity 1, competitors var(--grey-500) at 0.72
Labels inside cell if cell width > 40 AND height > 20:
  Brand: 4-char, font-size Math.min(11, cellWidth * 0.22), fill: var(--grey-900)
  Value: Baskerville italic, font-size Math.min(10, cellWidth * 0.20), fill: var(--grey-900)
No axes, no labels outside cells
```

---

### 9 · Treemap bar
**Metrics:** `bss` (snapshot, trend complement) · `sov` (snapshot)

Single full-width row. Width of each block = brand's share. Height: `72px`.

```
Total width: innerWidth (full card width)
Gap between blocks: 3px
Block fills:
  Illy: var(--primary-500)
  Competitors sorted desc: var(--grey-500) at opacity 1.0, 0.82, 0.65, 0.50, 0.38
Border-radius: 4px on leftmost and rightmost blocks only (rx=4)
Labels inside block if segW > 44px:
  Brand: 4-char, font-size Math.min(11, segW * 0.20), fill: var(--grey-900)
  Value: Baskerville italic, font-size Math.min(12, segW * 0.20), fill: var(--grey-900)
If segW ≤ 44px: omit label (no cramming)
```

---

### 10 · Bubbles
**Metrics:** `sov` (snapshot, alternative) · `bss` (snapshot, alternative)

```
d3.pack().size([width, height]).padding(4)
Circle fills: Illy var(--primary-500) opacity 1, competitors var(--grey-500) opacity 0.72
Labels inside if r > 14px: brand 4-char, fill: var(--grey-900)
Values inside if r > 24px: Baskerville italic %, fill: var(--grey-900)
No axes
```

---

### 11 · Inset bubble
**Metrics:** `sov` · `bss` · `eqr` · `ebl` · `nps` (alternative)

**Key mechanic:** outer circle = reference value (100 or max). Inner filled circle =
actual value. Area-proportional encoding: `r_inner = R_outer × √(value / reference)`.

```
Outer circle: r = min(54, slot/2 - 10), fill: none, stroke: var(--grey-700), stroke-width: 1
Inner circle: fill: brand colour, opacity: 1 (Illy) or 0.78 (competitors)
              minimum r = 2.5px regardless of value
Reference label top-right: "○ = [reference][unit]", 8px Tableau Light, var(--grey-700)
Brand label below outer circle: 9–10px Tableau Light uppercase

Value label placement:
  If r_inner ≥ 10px: label inside inner circle, fill: var(--grey-900)
  If r_inner < 10px (tiny): dashed leader line from inner circle top to above outer circle,
    label above outer circle in brand colour, 9.5px Baskerville italic

Reference values by metric:
  sov, bss: reference = 100 (percentage of total)
  eqr, ebl: reference = 1.0 (displayed as 0–100%)
  nps:      reference = 100 (people out of 100 respondents)
```

Sort order: Illy first (leftmost), then competitors sorted desc by value.

---

### 12 · Progress ring
**Metrics:** `eqr` (alternative) · `nps` (alternative) · `sov` (alternative)

```
Full circle ring: innerRadius = R - strokeWidth, outerRadius = R
Track arc: full 360°, fill: var(--grey-700), opacity 0.3
Fill arc: starts at -π/2 (12 o'clock), fills clockwise to -π/2 + (value/reference × 2π)
  cornerRadius = strokeWidth / 2 for rounded arc ends
  Illy: fill var(--primary-500). Competitors: fill var(--grey-500)
strokeWidth = max(8, R × 0.22)
Value label: Baskerville italic, centred in ring, font-size min(13, R × 0.30), brand colour
Brand label: below ring, Tableau Light 9–10px uppercase
Reference: same values as inset bubble
```

---

### 13 · Waffle chart
**Metrics:** `nps` (recommended) · `eqr` (alternative)

```
Grid: 10 × 10 = 100 dots
Dot size: 6px diameter, gap: 2px, step: 8px
Block per brand: 10 × 8 - 2 = 78px wide, 78px tall
Brand gap: distributed evenly across full width
Filled dots = Math.round((value / reference) × 100)
Fill order: top-left to bottom-right, row by row
Filled dot colour: var(--primary-500) (Illy), var(--grey-500) (competitors)
Empty dot colour: var(--grey-700), opacity 0.5
Brand label: below block, 9–10px Tableau Light uppercase, brand colour
Reference values: nps = 100 (promoters out of 100), eqr = 1.0
```

For `nps`: filled dots = promoters (brand colour), empty = passives + detractors.
For `eqr`: filled dots = quality engagement ratio × 100.

---

### 14 · Horizontal bar
**Metrics:** `cra` (recommended) · `sstsr` (recommended) · `dvtr` (alternative)

```
Layout: horizontal, brands as rows, sorted desc by value
Illy always at top row regardless of rank, separated by 0.5px rule from competitors
Row height: max 32px using scaleBand padding 0.25
Bar height: min(bandwidth, 20px), vertically centred in row, rx=2
Illy bar: var(--primary-500)
Competitors: var(--grey-500)
Brand label: right-aligned at x = -8, Tableau Light 9–10px uppercase,
  var(--primary-500) (Illy) or var(--grey-100) (competitors)
Value label: right of bar end + 8px, Baskerville italic 11px, brand colour
Benchmark / target rule: dashed 1px var(--grey-300), "BENCHMARK" 8.5px label above
X-axis: scaleLinear, tickSize = -rowCount × rowHeight (gridlines), Tableau Light 9px
```

For `cra`: unit = `%`, benchmark = industry average if available.
For `sstsr`: unit = `''` (ratio 0–1), benchmark = category average.
For `dvtr`: unit = `''`, scale [0, 5], benchmark = category average ~3.2.

---

### 15 · Vertical bar stacked
**Metrics:** `eqr` (recommended) · `ebl` (recommended)

Two segments per brand: **active** (bottom) and **passive** (top in visual terms,
but note: passive segment is rendered first / at y=0 in SVG since bars grow upward from baseline).

Wait — correct rendering order: passive occupies the upper portion (y=0 to y=pH where pH=y(active_ratio)), active occupies the lower portion (y=pH to y=iH). The active segment should visually be at the bottom.

Actually re-check: with scale y=scaleLinear([0,1] → [iH,0]), the active segment is the one that matters. Stack as:
- passive rect: x=bx, y=0, height=y(activeRatio) — the TOP portion
- active rect: x=bx, y=y(activeRatio), height=iH-y(activeRatio) — the BOTTOM portion
- separator line: horizontal 2px var(--grey-900) at y=y(activeRatio)

```
Illy active:   var(--primary-500)
Illy passive:  var(--primary-700)  #E0697D
Comp active:   var(--grey-500)
Comp passive:  var(--grey-700)
Separator:     stroke: var(--grey-900), stroke-width: 2
Y-axis: 0–100%, 4 ticks, gridlines, Tableau Light 9px
X-axis: brand names, no domain line
```

Data shape:
```js
// eqr: active = quality engagements / total, passive = 1 - active
// ebl: active = beyond-likes / total, passive = 1 - active
[{ brand: 'Illy', active: 0.082, passive: 0.918 }, ...]
```

---

### 16 · Scale · Figma
**Metrics:** `sop` (recommended) · `bt` (recommended) · `dvtr` (alternative)

Matches Figma Image 4 treatment. One row per brand, filled horizontal bar, brand
name above the bar, Baskerville score to the right.

```
Per brand row:
  Brand name: above-left, Tableau Light, 8.5px (Illy) 8px (others), uppercase,
    var(--primary-500) (Illy) or var(--grey-500) (competitors)
  Track: full width minus score column (52px), height: 34px, rx: 4,
    background: var(--grey-900)
  Filled bar: centred vertically in track (3px padding each side → 28px bar height),
    rx: 4, no gradient — solid fill only
    Illy: var(--primary-500). Competitors: var(--grey-500)
  Benchmark marker: 2.5px line, var(--grey-100), spanning full track height
    positioned at x = scale(benchmarkValue)
  Score: 15px Baskerville italic, brand colour, right-aligned in 52px column
  Row gap: 5px between brands
  Axis: at bottom, 8px Tableau Light, var(--grey-700), tick values only (no line)

Domain / benchmark per metric:
  sop: domain [2.5, 3.5], benchmark = category average ~3.0
  bt:  domain [2.7, 3.2], benchmark = target score = 3.5
  dvtr: domain [0, 5], benchmark = category average ~3.2
```

Sort rows: highest value at top. Illy always rendered at top regardless of rank,
separated by a 0.5px var(--grey-700) rule from the first competitor.

---

### 17 · Multi-scale dots
**Metrics:** `ba` (recommended)

**Requires `guardBaAttributes(data)` before rendering.**

```
One row per brand (Illy first, others sorted by overall avg desc)
Per row:
  Track line: thin 0.5px var(--grey-700) spanning x domain
  Attribute dots: one circle per attribute, positioned at x=scale(attrValue)
    Illy dots: r=6, fill: var(--primary-500), stroke: var(--grey-800) 1.5px
    Comp dots:  r=5, fill: var(--grey-500), opacity 0.85, stroke: var(--grey-800) 1.5px
  Connector line: thin horizontal line connecting min to max dot per row,
    stroke: brand colour, opacity 0.2
  Brand label: right-aligned at x = -8, Tableau Light 9–10px uppercase

Column header labels: attribute names abbreviated (3 chars), 7.5px Tableau Light,
  var(--grey-700), above Illy row only (reduces clutter)
X-axis: scaleLinear [2.5, 5], gridlines at tick positions, 9px var(--grey-700)
Attributes: Trust, Innovation, Value, Style, Heritage
```

Data shape:
```js
[{ brand: 'Illy', Trust: 3.8, Innovation: 3.2, Value: 2.9, Style: 4.1, Heritage: 4.4 }, ...]
```

---

### 18 · Dot matrix
**Metrics:** `nps` (alternative to waffle)

```
5 brands × 100 dots each (10 × 10 grid per brand)
Dot: circle r = 2.5px, gap = 2px, step = 7px
Block width: 10 × 7 - 2 = 68px
Brand gap: evenly distributed (FW - 5×68) / 4

Colour by dot index (i = 0..99):
  i < nPromoters:                   brand colour (Illy: var(--primary-500), comp: var(--grey-500))
  nPromoters ≤ i < nPromoters+nPass: var(--grey-700), opacity 0.5
  i ≥ nPromoters + nPass:            var(--primary-900)  #D73F58

Brand label: below block, 9–10px Tableau Light uppercase, brand colour

nPromoters = Math.round(d.promoterFraction × 100)
nPass = Math.round(d.passiveFraction × 100)
```

---

### 19 · Small multiples
**Metrics:** `bss` (context) · `sov` (context) · `svt` (context)

Provides per-brand trend context. Five mini panels in a single row.

```
5 equal-width panels across full width
Each panel:
  Background rect: var(--grey-900), rx=3, padding 4px
  Area fill: brand colour, fill-opacity: 0.5 (Illy), 0.28 (competitors)
  Line: brand colour, stroke-width: 1.5 (Illy), 1 (competitors), curveMonotoneX
  No axes, no labels inside panels
  Brand name: below panel, 9–10px Tableau Light uppercase, brand colour
Y scale: SAME domain for all panels — do not normalise per-brand
Curve: d3.curveMonotoneX
```

---

### 20 · Arc
**Metrics:** `nps` (Illy callout)

**Run `guardNpsRecalculated(data)` before rendering. If guard returns false, show
empty state with warning message. Do NOT deploy until NPS recalculation is confirmed.**

When data IS valid (true NPS scores, -100 to +100):

```
Semicircle geometry:
  startAngle: -π/2 (9 o'clock, left endpoint)
  endAngle:   +π/2 (3 o'clock, right endpoint)
  Arc passes through top (12 o'clock) — correct gauge orientation

Track arc: fill: var(--grey-700), opacity 0.3, full 360° ring
           innerRadius = R - strokeWidth, outerRadius = R

Fill arc: fill: var(--primary-500), solid, no gradient
          startAngle = -π/2
          endAngle = -π/2 + (value + 100) / 200 × π   [maps -100..+100 → 0..π]
          cornerRadius = strokeWidth / 2
          strokeWidth = max(20, R × 0.18)

Centre value: Baskerville italic, 52px, var(--primary-500), centred in arc
Brand label: "ILLY COFFEE", 10px Tableau Light, letter-spacing 0.08em, var(--grey-500)

Endpoint labels using D3 arc convention (NOT Math.cos/sin directly):
  left  x = cx + sin(-π/2) × (R + 12)   →   cx - R - 12
  left  y = cy - cos(-π/2) × R           →   cy
  right x = cx + sin(+π/2) × (R + 12)   →   cx + R + 12
  right y = cy - cos(+π/2) × R           →   cy
  Text: "−100" (left, text-anchor: end) and "+100" (right)

Competitor list (right panel, x = width × 0.64):
  Per competitor: 4px circle var(--grey-500), brand name 10px var(--grey-300),
  NPS score Baskerville italic 12px var(--grey-500), right-aligned
```

---

## Instance configurations

One file per metric in `src/instances/`. Each calls the relevant chart module
with metric-specific config.

```js
// src/instances/tam.js
import { initBAN } from '../charts/ban.js';
initBAN('chart-tam', {
  value: null,              // fetched from Tableau
  prefix: '$', unit: 'B',
  label: 'TOTAL ADDRESSABLE MARKET',
  sourceWorksheet: 'TAM_Data',
});

// src/instances/cagr.js
import { initBAN } from '../charts/ban.js';
initBAN('chart-cagr', {
  value: null, unit: '%', directional: true,
  label: 'COMPOUND ANNUAL GROWTH RATE',
  sourceWorksheet: 'Market_Data',
});

// src/instances/svt.js
import { initLine } from '../charts/line.js';
initLine('chart-svt', {
  straight: true,           // d3.curveLinear
  yFormat: '+.0%',
  unit: 'MoM % change',
  sourceWorksheet: 'Search_Volume',
  guard: 'svtZeros',
});

// src/instances/sop.js
import { initScaleFigma } from '../charts/scale-figma.js';
initScaleFigma('chart-sop', {
  domain: [2.5, 3.5], benchmark: 3.0, unit: '',
  label: 'SHARE OF PREFERENCE',
  sourceWorksheet: 'Survey_Data',
  guard: 'likertNotPercent',  // warn if data is Likert means
});

// src/instances/cra.js
import { initHBar } from '../charts/hbar.js';
initHBar('chart-cra', {
  unit: '%', benchmark: null,
  label: 'CONVERSION RATE ANALYSIS',
  sourceWorksheet: 'Conversion_Data',
});

// src/instances/bss.js — two instances: trend + snapshot
import { initArea100 }    from '../charts/area-100.js';
import { initTreemapBar } from '../charts/treemap-bar.js';
initArea100('chart-bss-trend', {
  label: 'BRANDED SEARCH SHARE · TREND',
  sourceWorksheet: 'Search_Share',
  brush: true,
});
initTreemapBar('chart-bss-snapshot', {
  label: 'BRANDED SEARCH SHARE · SNAPSHOT',
  sourceWorksheet: 'Search_Share',
  period: 'latest',
});

// src/instances/dvtr.js
import { initBANs } from '../charts/bans.js';
initBANs('chart-dvtr', {
  label: 'DISTINCTIVE VERBAL TONE RECOGNITION',
  sourceWorksheet: 'Survey_Data',
});

// src/instances/sstsr.js
import { initHBar } from '../charts/hbar.js';
initHBar('chart-sstsr', {
  unit: '', benchmark: 0.2,
  label: 'SCROLL-STOP / THUMB-STOP RATIO',
  sourceWorksheet: 'Social_Data',
});

// src/instances/eqr.js
import { initVBarStacked } from '../charts/vbar-stacked.js';
initVBarStacked('chart-eqr', {
  activeKey: 'activeEngagements',
  passiveKey: 'passiveEngagements',
  label: 'ENGAGEMENT QUALITY RATIO',
  sourceWorksheet: 'Engagement_Data',
});

// src/instances/sov.js — two instances: trend + snapshot
import { initArea100 }    from '../charts/area-100.js';
import { initTreemapBar } from '../charts/treemap-bar.js';
initArea100('chart-sov-trend', {
  label: 'SHARE OF VOICE · TREND',
  sourceWorksheet: 'Social_Share',
});
initTreemapBar('chart-sov-snapshot', {
  label: 'SHARE OF VOICE · SNAPSHOT',
  sourceWorksheet: 'Social_Share',
  period: 'latest',
});

// src/instances/vom.js
import { initLine } from '../charts/line.js';
initLine('chart-vom', {
  straight: false,          // d3.curveMonotoneX
  yFormat: '+.0%',
  unit: 'MoM velocity',
  sourceWorksheet: 'Mentions_Data',
});

// src/instances/nps.js
import { initWaffle } from '../charts/waffle.js';
initWaffle('chart-nps', {
  reference: 100,
  valueKey: 'promoterCount',
  label: 'PROMOTERS · PER 100 RESPONDENTS',
  sourceWorksheet: 'Survey_Data',
  guard: 'npsRecalculated',
});
// Arc instance is built but not deployed:
// import { initArc } from '../charts/arc.js';
// initArc('chart-nps-callout', { guard: 'npsRecalculated', ... });

// src/instances/ba.js
import { initMultiScale } from '../charts/multiscale.js';
initMultiScale('chart-ba', {
  attributes: ['Trust', 'Innovation', 'Value', 'Style', 'Heritage'],
  domain: [2.5, 5],
  label: 'BRAND AFFINITY',
  sourceWorksheet: 'Affinity_Data',
  guard: 'baAttributes',
});

// src/instances/ebl.js
import { initVBarStacked } from '../charts/vbar-stacked.js';
initVBarStacked('chart-ebl', {
  activeKey: 'beyondLikes',
  passiveKey: 'likesOnly',
  label: 'BRAND ENGAGEMENT BEYOND LIKES',
  sourceWorksheet: 'Engagement_Data',
});

// src/instances/bt.js
import { initScaleFigma } from '../charts/scale-figma.js';
initScaleFigma('chart-bt', {
  domain: [2.7, 3.2], benchmark: 3.5, unit: '',
  label: 'BRAND TRUST',
  sourceWorksheet: 'Survey_Data',
  guard: 'starbucksPresent',  // warn if Starbucks missing
});
```

---

## Deliverables checklist

- [ ] `tokens.css` — all CSS custom properties
- [ ] `lib/scaffold.js` — base SVG + ResizeObserver
- [ ] `lib/selection.js` — event bus + Tableau filter
- [ ] `lib/tooltip.js` — shared tooltip component
- [ ] `lib/data-guards.js` — all validation functions
- [ ] 20 chart modules in `charts/`
- [ ] 15 instance configs in `instances/` (bss and sov have 2 instances each = 17 total)
- [ ] All charts tested at 200×600px (compact Tableau wide-ratio mode)
- [ ] Selection propagates across all charts on same dashboard
- [ ] `arc.js` built but instance commented out pending NPS recalculation
- [ ] `ba` multi-scale shows placeholder if attribute columns absent
- [ ] `svt` renders with Q4 warning annotation if zero values detected

---

## Known data issues — handle before rendering

| Metric | Issue | Guard | Action |
|---|---|---|---|
| `nps` | Likert means (2.52–3.24) not NPS scores | `guardNpsRecalculated` | Use waffle/BANs; block arc |
| `ba` | 4 of 5 brands identical value 4.060 — pipeline error | `guardBaAttributes` | Render warning, no chart |
| `svt` | Q4 values all zero — pipeline unpopulated | `guardSvtZeros` | Render with annotation |
| `bt` | Starbucks missing | `guardStarbucksPresent` | Warn in console, render remaining |
| `sop` `dvtr` `bt` | Likert means served as % | `guardLikertNotPercent` | Warn, skip render |
