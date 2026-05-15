# Brand Intelligence Dashboard · Chart Library
## Implementation specification for Claude Code

Read every section before writing any code.
All hex values, formulas, px values, and data guards are mandatory.

---

## 1 · Metric → chart mapping

| ID | Metric | Primary chart | Secondary / alternative |
|---|---|---|---|
| `tam` | Total Addressable Market | BAN | — |
| `cagr` | Compound Annual Growth Rate | BAN | — |
| `svt` | Search Volume & Trends | Straight line | Smooth line |
| `sop` | Share of Preference | Scale · Figma | Slope chart |
| `cra` | Conversion Rate Analysis | Horizontal bar | — |
| `bss` | Branded Search Share | Stacked 100% area (trend) + Treemap bar (snapshot) | Slope, Stream, Small multiples |
| `dvtr` | Distinctive Verbal Tone Recognition | BANs | Horizontal bar, Scale · Figma |
| `sstsr` | Scroll-stop / Thumb-stop Ratio | Horizontal bar | — |
| `eqr` | Engagement Quality Ratio | Vertical bar stacked | Inset bubble, Progress ring, Waffle |
| `sov` | Share of Voice | Stacked 100% area (trend) + Treemap bar (snapshot) | Bubbles, Inset bubble, Stream |
| `vom` | Velocity of Mentions | Smooth line | Straight line |
| `nps` | Brand Loyalty / NPS | Waffle chart | Dot matrix, Inset bubble, Arc (Illy callout — HOLD) |
| `ba` | Brand Affinity | Multi-scale dots | — |
| `ebl` | Brand Engagement beyond likes | Vertical bar stacked | Inset bubble |
| `bt` | Brand Trust | Scale · Figma | BANs |

`bss` and `sov` each require two chart instances: one trend view, one snapshot view.

---

## 2 · Project setup

```bash
npm init -y && npm install d3@7
```

```
src/
  tokens.css
  lib/
    scaffold.js       # SVG shell + ResizeObserver
    selection.js      # cross-chart event bus + Tableau filter
    tooltip.js        # shared hover tooltip
    data-guards.js    # pre-render validation
  charts/
    ban.js            # tam, cagr
    bans.js           # dvtr · bt (alt) · nps (alt)
    line.js           # svt (straight) · vom (smooth) — shared module, config flag
    slope.js          # bss · sov · sop (alt)
    area-100.js       # bss · sov (trend)
    stream.js         # bss · sov (alt)
    treemap.js        # bss · sov · tam (snapshot)
    treemap-bar.js    # bss · sov (snapshot)
    bubbles.js        # sov · bss (alt snapshot)
    inset-bubble.js   # sov · bss · eqr · ebl · nps (alt)
    progress-ring.js  # eqr · nps · sov (alt)
    waffle.js         # nps · eqr (alt)
    hbar.js           # cra · sstsr · dvtr (alt)
    vbar-stacked.js   # eqr · ebl
    scale-figma.js    # sop · bt · dvtr (alt)
    multiscale.js     # ba
    dot-matrix.js     # nps (alt)
    small-multiples.js# bss · sov · svt (context)
    arc.js            # nps Illy callout — build, do NOT deploy
  instances/
    tam.js  cagr.js  svt.js  sop.js  cra.js  bss.js
    dvtr.js sstsr.js eqr.js  sov.js  vom.js  nps.js
    ba.js   ebl.js   bt.js
```

---

## 3 · Design tokens  `src/tokens.css`

```css
:root {
  /* Backgrounds */
  --grey-900: #1A1A1A;   /* page background */
  --grey-800: #2A2828;   /* card surface */
  --grey-700: #4D4D4D;   /* passive fills · empty dots · track fills */
  --grey-500: #808080;   /* competitor fills and lines */
  --grey-300: #B3B3B3;   /* axis labels · secondary text */
  --grey-100: #E6E6E6;   /* primary body text */

  /* Illy brand — primary only on Illy marks, never on competitors */
  --primary-500: #E994A2;  /* Illy main fill */
  --primary-700: #E0697D;  /* Illy passive segment (stacked bar top) */
  --primary-900: #D73F58;  /* negative semantic: detractors, decline */

  /* DO NOT USE any other pink/red. DO NOT USE gradients anywhere. */
}
```

---

## 4 · Typography

| Role | Size | Family | Weight | Colour token |
|---|---|---|---|---|
| BAN headline | 80px | Baskerville | Italic | `--primary-500` |
| BANs · Illy value | 56px | Baskerville | Italic | `--primary-500` |
| BANs · competitor values | 32px | Baskerville | Italic | `--grey-500` |
| Arc / ring centre value | 52px | Baskerville | Italic | `--primary-500` |
| Scale · Figma score | 14–15px | Baskerville | Italic | brand colour |
| Direct bar / dot labels | 11–12px | Baskerville | Italic | brand colour |
| Brand name labels | 9–10px | Tableau Light | Regular | `--primary-500` (Illy) · `--grey-100` (comps) |
| Axis tick values | 9px | Tableau Light | Regular | `--grey-500` |
| Benchmark labels | 8.5px | Tableau Light | Regular | `--grey-700` |
| Chart sub-label / note | 8px | Tableau Light | Italic | `--grey-700` |

All brand name labels: UPPERCASE, letter-spacing 0.06em.
Axis tick labels: as-is (not uppercase).

---

## 5 · Layout constants

```js
// Used by every chart module
const MARGIN         = { top: 20, right: 110, bottom: 36, left: 52 };
const MARGIN_COMPACT = { top:  8, right:  80, bottom: 20, left: 36 };
const MIN_HEIGHT     = 120;   // below this: render "Container too small" only
const isCompact      = h => h < 220;
const yTickCount     = h => h > 300 ? 5 : h > 150 ? 3 : 2;

// SVG always responsive — never fixed px on svg element
// Read dimensions at render time:
const { width, height } = container.getBoundingClientRect();
```

---

## 6 · Cross-cutting rules

### 6.1 Z-order
Sort data array before binding so Illy appends last and renders on top:
```js
const sorted = [
  ...data.filter(d => d.brand !== 'Illy'),
  ...data.filter(d => d.brand === 'Illy'),
];
```
Apply this in every chart that renders per-brand marks.

### 6.2 Selection opacity
On `broadcastSelection(brand)`, transition all `[data-brand]` elements:
```js
duration: 180ms  easing: d3.easeCubicOut

opacity = !selected          → 1.0
opacity = brand === selected → 1.0
opacity = brand === 'Illy'   → 0.35   // Illy floor — never below this
opacity = other competitor   → 0.20
```
Click background (no brand target) → call `broadcastSelection(null)` → all return to 1.0.

### 6.3 Illy separator rule
For charts with horizontal brand rows (hbar, scale-figma, multiscale, slope labels):
insert `0.5px solid var(--grey-700)` rule between Illy row and the first competitor row.

### 6.4 Transition library
| Trigger | Duration | Easing | Property |
|---|---|---|---|
| Bar enter (height/width from baseline) | 600ms | `d3.easeCubicOut` | height or width |
| Line/area path enter (left→right reveal) | 800ms | `d3.easeLinear` | stroke-dashoffset |
| Circle/dot enter | 400ms + 40ms stagger | `d3.easeBackOut` | r from 0 |
| Arc fill enter | 800ms | `d3.easeCircleOut` | endAngle via attrTween |
| Text label enter | 300ms delay after marks | `d3.easeLinear` | opacity 0 → 1 |
| Selection state change | 180ms | `d3.easeCubicOut` | opacity |
| Hover ring appear | 80ms | `d3.easeLinear` | stroke-width 0 → 1.5 |
| Hover ring disappear | 120ms | `d3.easeLinear` | stroke-width 1.5 → 0 |
| Data update | 400ms | `d3.easeCubicInOut` | all positions and sizes |
| Resize re-render | 0ms — cancel any in-progress transitions first | — | — |

### 6.5 State templates (apply to every chart)

**Loading:**
```css
/* Replace chart body with shimmer rect of identical dimensions */
background: linear-gradient(90deg,
  var(--grey-800) 25%, var(--grey-700) 50%, var(--grey-800) 75%);
background-size: 200% 100%;
animation: shimmer 1.5s linear infinite;
border-radius: 4px;
```

**Empty (no data):**
SVG/HTML: centred text "No data available", 11px Tableau Light, `var(--grey-700)`.
BAN/BANs: render `—` at full headline size in `var(--grey-700)`.

**Too small (height < 120px):**
Clear SVG, render: centred text "Container too small", 11px, `var(--grey-700)`.

**Compact (height < 220px):**
- Switch to `MARGIN_COMPACT`
- `yTickCount` → 2
- Remove axis domain lines
- Bar height cap: 12px (was 20px)
- Line stroke-width: 1px Illy / 0.8px competitors
- Waffle/dot-matrix: reduce to 8×8 = 64 dots if height < 160px
- Scale · Figma: track height 24px, bar height 18px
- Small multiples: `cellH` = 52px

### 6.6 ResizeObserver
```js
const ro = new ResizeObserver(() => {
  clearTimeout(render._debounce);
  render._debounce = setTimeout(render, 150);
});
ro.observe(container);
```

---

## 7 · Shared modules

### 7.1 `src/lib/scaffold.js`
```js
export function buildChart(containerId, drawFn) {
  const container = document.getElementById(containerId);
  const svg = d3.select(container).append('svg')
    .style('width','100%').style('height','100%').style('display','block');
  const g = svg.append('g');

  function render() {
    const { width, height } = container.getBoundingClientRect();
    if (height < 120) {
      svg.selectAll('*').remove();
      svg.append('text')
        .attr('x', width/2).attr('y', height/2)
        .attr('text-anchor','middle')
        .attr('fill','var(--grey-700)')
        .attr('font-size', 11)
        .attr('font-family','Arial,sans-serif')
        .text('Container too small');
      return;
    }
    const M = height < 220 ? MARGIN_COMPACT : MARGIN;
    const iW = width  - M.left - M.right;
    const iH = height - M.top  - M.bottom;
    g.attr('transform', `translate(${M.left},${M.top})`);
    g.selectAll('*').remove();
    drawFn(g, svg, { width, height, M, iW, iH });
  }

  const ro = new ResizeObserver(() => {
    clearTimeout(render._t);
    render._t = setTimeout(render, 150);
  });
  ro.observe(container);
  return { render, svg, g };
}
```

### 7.2 `src/lib/selection.js`
```js
let _selected = null;
const _listeners = new Set();

export function broadcastSelection(brand) {
  _selected = (_selected === brand) ? null : brand;
  _listeners.forEach(fn => fn(_selected));
  try {
    const ws = tableau.extensions.dashboardContent.dashboard.worksheets
      .find(w => w.name === window.__CHART_CONFIG__?.sourceWorksheet);
    _selected
      ? ws.applyFilterAsync('Brand', [_selected], FilterUpdateType.Replace)
      : ws.clearFilterAsync('Brand');
  } catch (_) { /* no Tableau context — visual-only selection */ }
}

export const onSelectionChange = fn => {
  _listeners.add(fn);
  return () => _listeners.delete(fn);
};

export const getSelected = () => _selected;

export function applySelectionOpacity(selection, selected) {
  selection.selectAll('[data-brand]')
    .transition().duration(180).ease(d3.easeCubicOut)
    .attr('opacity', function() {
      const b = this.getAttribute('data-brand');
      if (!selected) return 1;
      if (b === selected) return 1;
      if (b === 'Illy') return 0.35;
      return 0.2;
    });
}
```

### 7.3 `src/lib/tooltip.js`
```js
export const tooltip = {
  el: null,
  init() {
    this.el = document.createElement('div');
    Object.assign(this.el.style, {
      position:'fixed', background:'var(--grey-800)',
      border:'1px solid var(--grey-700)', borderRadius:'6px',
      padding:'10px 14px', pointerEvents:'none',
      zIndex:'9999', maxWidth:'200px', display:'none',
      fontFamily:'Arial,sans-serif',
    });
    document.body.appendChild(this.el);
  },
  show(event, header, rows) {
    // rows: [{ brand: string, label: string, value: string }]
    const hdr = `<div style="font-size:11px;color:var(--grey-500);
      letter-spacing:.06em;text-transform:uppercase;
      border-bottom:1px solid var(--grey-700);
      padding-bottom:6px;margin-bottom:6px">${header}</div>`;
    const body = rows.map(r => `
      <div style="display:flex;justify-content:space-between;gap:16px;padding:2px 0">
        <span style="display:flex;align-items:center;gap:6px;font-size:11px;
          color:var(--grey-300)">
          <span style="width:6px;height:6px;border-radius:50%;
            background:${r.brand==='Illy'
              ?'var(--primary-500)':'var(--grey-500)'}"></span>
          ${r.label}</span>
        <span style="font-size:11px;color:${r.brand==='Illy'
          ?'var(--primary-500)':'var(--grey-100)'}">${r.value}</span>
      </div>`).join('');
    this.el.innerHTML = hdr + body;
    this.el.style.display = 'block';
    this._position(event);
  },
  _position(event) {
    const { innerWidth:W, innerHeight:H } = window;
    const { width:w, height:h } = this.el.getBoundingClientRect();
    const x = event.clientX + 12, y = event.clientY - h/2;
    this.el.style.left = (x + w > W ? x - w - 24 : x) + 'px';
    this.el.style.top  = Math.max(8, Math.min(H - h - 8, y)) + 'px';
  },
  hide() { this.el.style.display = 'none'; },
};
```

### 7.4 `src/lib/data-guards.js`
```js
export function guardLikertNotPercent(data, metricId) {
  if (data.some(d => d.value > 1 && d.value <= 5)) {
    console.warn(`[${metricId}] Values look like Likert means (1–5), not percentages. Skipping render.`);
    return false;
  }
  return true;
}

export function guardStarbucksPresent(data, metricId) {
  if (!data.some(d => d.brand === 'Starbucks'))
    console.warn(`[${metricId}] Starbucks missing from dataset.`);
  // warn only — do not block render
}

export function guardSvtZeros(data) {
  const q4zero = data.filter(d =>
    ['Sep','Oct','Nov','Dec'].includes(d.month) && d.value === 0);
  if (q4zero.length > 0)
    console.warn('[svt] Q4 values are zero — pipeline unpopulated. Rendering with annotation.');
  return q4zero.length > 0; // returns true if annotation needed
}

export function guardNpsRecalculated(data) {
  if (data.some(d => d.nps > 5)) return true;
  console.warn('[nps] Values appear to be Likert means, not true NPS (-100 to +100). Arc blocked.');
  return false;
}

export function guardBaAttributes(data) {
  if (!data[0]?.Trust) {
    console.warn('[ba] Attribute columns missing (need Trust, Innovation, Value, Style, Heritage).');
    return false;
  }
  return true;
}
```

---

## 8 · Chart specifications

---

### Chart 01 · BAN
**Metrics:** `tam` · `cagr`
Pure HTML, no SVG, no D3.

**Dimensions**
```
container:   min-height 96px, padding 20px 24px
             background var(--grey-800), border-radius 8px
             border 1px solid var(--grey-700), position relative
```

**Marks**
```
label:   font-family Tableau Light, font-size 10px, text-transform uppercase
         letter-spacing 0.10em, color var(--primary-500), margin-bottom 4px

value:   font-family Baskerville, font-style italic, font-size 80px
         color var(--primary-500), line-height 0.9
         unit shown inline (e.g. "$14.4B" or "1.9%")

period:  font-family Tableau Light, font-size 9px, color var(--grey-700)
         position absolute, bottom 12px, right 16px
```

**`cagr` variant only:**
Prepend `▲` (color var(--primary-500)) for positive values.
Prepend `▼` (color var(--primary-900)) for negative values.

**States**
```
loading: shimmer rect, height 60px, width 70%, border-radius 4px
empty:   render — at 80px Baskerville italic, color var(--grey-700)
```

**No transitions, no interactions — display only.**

```js
// Data shape
{ value: 14.425, unit: 'B', prefix: '$', period: '2025',
  label: 'TOTAL ADDRESSABLE MARKET' }
```

---

### Chart 02 · BANs
**Metrics:** `dvtr` (primary) · `bt` (alt) · `nps` (alt, only if NPS recalc fails)
Pure HTML, no SVG, no D3. Typography IS the chart.

**Dimensions**
```
container:   display flex, align-items flex-end, flex-wrap wrap, gap 0
             padding 16px 20px, background var(--grey-800)
             border-radius 8px, border 1px solid var(--grey-700)
```

**Marks — Illy block**
```
wrapper:  padding-right 24px, margin-right 24px
          border-right 1px solid var(--grey-700)

label:    font-family Tableau Light, font-size 7.5px, text-transform uppercase
          letter-spacing 0.12em, color var(--primary-500), margin-bottom 4px

value:    font-family Baskerville, font-style italic, font-size 56px
          color var(--primary-500), line-height 0.92
          data-brand="Illy" attribute on this element
```

**Marks — competitor blocks (sorted desc by value)**
```
wrapper:  padding-right 20px

label:    font-family Tableau Light, font-size 7px, text-transform uppercase
          letter-spacing 0.10em, color var(--grey-500), margin-bottom 4px

value:    font-family Baskerville, font-style italic, font-size 32px
          color var(--grey-500), line-height 0.92
          data-brand="{brand}" attribute on this element
```

**States**
```
loading Illy:  shimmer rect 56px tall, width 120px, border-radius 4px
loading comps: shimmer rect 32px tall, width 80px
empty:         — at respective sizes, color var(--grey-700)
```

**Interaction:** `data-brand` attributes enable `applySelectionOpacity`. No click on BAN itself.

```js
// Data shape
[{ brand: 'Illy', value: 3.04 }, { brand: 'Peets', value: 3.34 }, ...]
```

---

### Chart 03 · Straight line
**Metrics:** `svt` (primary) · `vom` (alt)

**Dimensions**
```
margin: top 12, right 58, bottom 28, left 38
height: flexible (Tableau container), typical 180–240px
```

**Marks**
```
Illy line:      stroke var(--primary-500), stroke-width 2px, fill none
comp lines:     stroke var(--grey-500), stroke-width 1px, fill none
curve:          d3.curveLinear (straight segments between data points)
zero rule:      y=0 line, stroke var(--grey-500) stroke-width 1px opacity 0.55
                only render if y=0 falls within iH bounds
end dots:       r=3px Illy / r=2px comps, fill brand colour
                at both first and last data point per brand
svt annotation: if guardSvtZeros returns true, append text
                "⚠ Q4 data pending", x=iW×0.72, y=16
                9px Tableau Light, color var(--grey-700)
```

**Axes**
```
x-axis: d3.scalePoint, d3.axisBottom
        domain: array of month strings
        tickSize 0, tickPadding 6, no domain line
        9px Tableau Light, color var(--grey-500)

y-axis: d3.scaleLinear, d3.axisLeft
        domain: [d3.min(all values)×1.1, d3.max(all values)×1.1]
        tickSize -iW (full-width gridlines)
        gridlines: stroke var(--grey-700) opacity 0.35, stroke-dasharray 2 3
        tick format: d3.format('+.0%'), 9px Tableau Light var(--grey-500)
        remove domain line: ax.select('.domain').remove()
```

**Labels (right of chart, in right margin)**
```
brand 4-char uppercase, 9px Tableau Light, brand colour
x = iW + 6, y = y(lastValue) + 4, text-anchor start
collision: sort brands by last value desc
  if label within 12px of previous placed label → offset by 12px
```

**Enter transitions**
```
1. Measure path length: node.getTotalLength()
2. Set stroke-dasharray = length, stroke-dashoffset = length
3. Transition stroke-dashoffset → 0, 800ms d3.easeLinear
4. End dots: r from 0, 400ms d3.easeBackOut, stagger 40ms per brand
5. Labels: opacity 0→1, 300ms, begin 500ms after line starts
```

**Compact:** stroke-width 1px Illy / 0.8px comps. Omit end dots if compact.

```js
// Data shape — wide format
[{ month: 'Feb', Starbucks: 0.189, Nespresso: -0.017,
   Peets: 0.07, Lavazza: 0.08, Illy: 0.025 }, ...]
```

---

### Chart 04 · Smooth line
**Metrics:** `vom` (primary) · `svt` (alt)

Identical to Chart 03 with one change:
```
curve: d3.curveMonotoneX   (was d3.curveLinear)
```

Implement in the same `line.js` module via `config.straight` flag:
```js
const curve = config.straight ? d3.curveLinear : d3.curveMonotoneX;
```

All dimensions, colours, axes, labels, transitions, and data shape identical to Chart 03.

---

### Chart 05 · Slope chart
**Metrics:** `bss` (primary) · `sov` (alt) · `sop` (alt, two survey periods)

**Dimensions**
```
margin: top 32, right 90, bottom 12, left 90
height: flexible, typical 200–240px
```

**Marks**
```
left pole:   x=0, y1=0, y2=iH
             stroke var(--grey-700), stroke-width 0.5px

right pole:  x=iW, y1=0, y2=iH
             stroke var(--grey-700), stroke-width 0.5px

period labels: above each pole, y=-14, text-anchor middle
               9px Tableau Light uppercase, color var(--grey-300)

connecting lines:
  Illy:  stroke var(--primary-500), stroke-width 2.5px
  comps: stroke var(--grey-500), stroke-width 1px
  data-brand attribute on each line

end dots:
  Illy:  r=5px, fill var(--primary-500)
  comps: r=4px, fill var(--grey-500)
  data-brand attribute on each dot
```

**Labels**
```
y-scale: d3.scaleLinear, domain [0, max(all v1, v2) × 1.1], no y-axis rendered

left labels (x=-9, text-anchor end):
  "{BRAND} {value}" — brand 4-char uppercase + space + formatted value
  10px (Illy) / 8.5px (comps) Tableau Light, brand colour

right labels (x=iW+9, text-anchor start):
  same format
  10px (Illy) / 8.5px (comps) Tableau Light, brand colour

collision detection (apply independently to left and right):
  sort brands by v1 (left) or v2 (right) desc
  track placed y positions in array
  if abs(placedY - newY) < 11 → newY = placedY + 11
  place from top to bottom
```

**Enter:** lines from x=0 to x=iW, 600ms d3.easeCubicOut. Dots r from 0 stagger 50ms.

**Compact:** left/right margins 70px. Labels: brand 4-char only, no value, font-size 8px.

```js
// Data shape
[{ brand: 'Illy', v1: 0.57, v2: 0.59 },
 { brand: 'Starbucks', v1: 82, v2: 83.9 }, ...]
// v1 = start period value, v2 = end period value
```

---

### Chart 06 · Stacked 100% area
**Metrics:** `bss` (trend) · `sov` (trend)

**Dimensions**
```
margin: top 10, right 52, bottom 28, left 32
        if config.brush: add 44px to bottom
height: flexible, typical 160–200px
```

**Stack**
```
d3.stack().keys(brandOrder).offset(d3.stackOffsetExpand)

brandOrder: sort brands by avg share desc → dominant brand index 0 (bottom of stack)
            Illy always index n-1 (top of stack), regardless of share
```

**Marks**
```
areas: d3.area().x(d => x(d.data.month)).y0(d => y(d[0])).y1(d => y(d[1]))
       .curve(d3.curveMonotoneX)

fills:
  Illy:    fill var(--primary-500), fill-opacity 0.78
  rank 1:  fill var(--grey-500), fill-opacity 0.75
  rank 2:  fill var(--grey-500), fill-opacity 0.60
  rank 3:  fill var(--grey-500), fill-opacity 0.45
  rank 4:  fill var(--grey-500), fill-opacity 0.32

boundaries: stroke var(--grey-900), stroke-width 0.5px on each path
data-brand attribute on each path element
```

**Axes**
```
x-axis: d3.scalePoint, month labels
        9px Tableau Light, color var(--grey-500), no domain line

y-axis: 0–100%, d3.format('.0%'), 3 ticks, tickSize -iW
        gridlines var(--grey-700) opacity 0.35, stroke-dasharray 2 3
        9px Tableau Light, color var(--grey-500), remove domain line
```

**Inline labels (right margin, 52px)**
```
brand 4-char uppercase, 9px Tableau Light, brand colour
x = iW + 5, y = y((last[0] + last[1]) / 2) + 4
only render if layer thickness at last point > 4% chart height
collision: ±8px offset if within 12px of another label
```

**Brush (if config.brush = true)**
```
d3.brushX below x-axis, height 28px, gap 8px from axis
background var(--grey-800) rect behind brush
mini sparkline per brand at 15% opacity inside brush strip
brush handle circles: fill var(--primary-500)
double-click on brush area → reset to full range
```

**Interactions**
```
click area layer → isolation: clicked layer fill-opacity 0.78,
                  all others fill-opacity 0.12 (Illy floor: 0.35)
click SVG background → restore all fill-opacities
```

**Enter:** layers fade in bottom→top, 400ms d3.easeCubicOut, 100ms stagger.

```js
// Data shape — wide format
[{ month: 'Jan', Starbucks: 82, Nespresso: 10.7,
   Peets: 6, Lavazza: 0.72, Illy: 0.57 }, ...]
```

---

### Chart 07 · Stream graph
**Metrics:** `bss` (alt) · `sov` (alt)

**Dimensions:** same as Chart 06, no right margin needed for labels.

**Stack**
```
d3.stack().keys(brandOrder).offset(d3.stackOffsetSilhouette)
brandOrder: same ranking as Chart 06
curve: d3.curveBasis
```

**Marks**
```
fills: same opacity cascade as Chart 06
boundaries: none (organic, no stroke between layers)
data-brand attribute on each path
```

**Axes**
```
x-axis only — months, 9px Tableau Light var(--grey-500)
no y-axis (silhouette offset makes y-values meaningless)
```

**Mid-point labels**
```
centred horizontally and vertically within each band at chart midpoint
text-anchor middle, fill var(--grey-900), 9px Tableau Light
only render if band height > 20px at midpoint
```

**Code comment (required):**
`// Stream silhouette distorts absolute values. Visual trend only. Pair with treemap-bar for precision.`

**Enter:** paths opacity 0→fill-opacity, 600ms d3.easeCubicOut, stagger 80ms bottom→top.

Same data shape as Chart 06.

---

### Chart 08 · Treemap
**Metrics:** `bss` (snapshot) · `sov` (snapshot) · `tam` (if SAM/SOM added)

**Dimensions:** full container width and height, no margin.

**Layout**
```
d3.treemap().size([width, height]).padding(2).round(true)
data: latest single period — extract most recent month from time series
sort: d3.treemap sorts by value desc automatically
```

**Marks**
```
rects:
  Illy:  fill var(--primary-500), opacity 1
  comps: fill var(--grey-500), opacity 0.72
  data-brand attribute
  cursor pointer, on click: broadcastSelection(brand)
```

**Cell labels (inside rect, only when space allows)**
```
condition: cellW > 40 AND cellH > 20

brand label:
  text brand.slice(0,4), font-family Tableau Light, text-transform uppercase
  font-size Math.min(11, cellW × 0.22)
  fill var(--grey-900), x=node.x0+5, y=node.y0+14

value label (additional condition: cellH > 30):
  text formatted value, font-family Baskerville, font-style italic
  font-size Math.min(10, cellW × 0.20)
  fill var(--grey-900), x=node.x0+5, y=node.y0+26
```

**Enter:** rects scale from centre (transform-origin centre of each cell),
500ms d3.easeCubicOut, stagger 30ms per cell (sorted by area desc).

```js
// Data shape — single period
[{ brand: 'Starbucks', value: 83.9 }, { brand: 'Illy', value: 0.59 }, ...]
```

---

### Chart 09 · Treemap bar
**Metrics:** `bss` (snapshot) · `sov` (snapshot)

One wide row. Each brand = a proportional block. Width = share.

**Dimensions**
```
height: 72px fixed
gap:    3px between blocks
no margin, no axes, no labels outside blocks
```

**Block layout**
```
sort: brands by value desc — dominant brand leftmost
      Illy not forced to any position; identified by colour only

usableW = containerWidth - gap × (nBrands - 1)
segW[i] = (brand.value / totalValue) × usableW

border-radius:
  leftmost block:  rx=4, ry=4 on left corners only
  rightmost block: rx=4, ry=4 on right corners only
  middle blocks:   rx=0
```

**Fills**
```
Illy:      fill var(--primary-500), opacity 1
comp rank 1: fill var(--grey-500), opacity 1.00
comp rank 2: fill var(--grey-500), opacity 0.82
comp rank 3: fill var(--grey-500), opacity 0.65
comp rank 4: fill var(--grey-500), opacity 0.50
comp rank 5: fill var(--grey-500), opacity 0.38
data-brand attribute on each rect
```

**Labels (inside block, fill var(--grey-900))**
```
only render if segW > 44px

brand: font-family Tableau Light, text-transform uppercase
       font-size Math.min(11, segW × 0.20)
       x=cx, y=H/2-5, text-anchor middle

value: font-family Baskerville, font-style italic
       font-size Math.min(12, segW × 0.20)
       x=cx, y=H/2+9, text-anchor middle

if segW ≤ 44px: render no label (no cramming)
```

**Enter:** blocks grow from x=0 to full widths, 600ms d3.easeCubicOut.
**Compact:** height 48px, font-size caps -2px.

```js
// Data shape — single period
[{ brand: 'Starbucks', value: 83.9 }, { brand: 'Illy', value: 0.59 }, ...]
```

---

### Chart 10 · Bubbles
**Metrics:** `sov` (snapshot, alt) · `bss` (snapshot, alt)

**Layout**
```
d3.pack().size([width, height]).padding(4)
data: single period
SVG: full container width and height
```

**Marks**
```
circles:
  Illy:  fill var(--primary-500), opacity 1
  comps: fill var(--grey-500), opacity 0.72
  data-brand attribute, cursor pointer, on click: broadcastSelection(brand)
```

**Labels (inside circle only — no external labels)**
```
brand (if r > 14px):
  text brand.slice(0,4), Tableau Light uppercase
  font-size Math.min(11, r × 0.32), fill var(--grey-900)
  y = cy - 4  (if value label also showing)
  y = cy + 4  (if no value label)

value (if r > 24px):
  text formatted %, Baskerville italic
  font-size Math.min(10, r × 0.26), fill var(--grey-900)
  y = cy + 10

if r ≤ 14px: render no label
```

**Enter:** r from 0, 400ms d3.easeBackOut, stagger 40ms per bubble.
**Compact:** padding 2, label thresholds r>10 (brand) / r>18 (value).

```js
// Data shape — single period
[{ brand: 'Starbucks', value: 80.66 }, { brand: 'Illy', value: 0.26 }, ...]
```

---

### Chart 11 · Inset bubble
**Metrics:** `sov` · `bss` · `eqr` · `ebl` · `nps` (alt)

Outer circle = reference (100 or max). Inner filled circle = actual value.
Encoding: `r_inner = R_outer × √(value / reference)` — area-proportional.

**Layout**
```
sort:   Illy → slot 0 (leftmost). Others sorted desc by value → slots 1–n
slotW:  containerWidth / nBrands
R:      Math.min(54, slotW/2 - 10)
cx[i]:  slotW × i + slotW / 2
cy:     R + 16   (16px from top for reference label)
SVG height: R×2 + 54
```

**Marks**
```
outer circle (context — no transition):
  r=R, fill none, stroke var(--grey-700), stroke-width 1px
  data-brand attribute

inner circle:
  r_inner = Math.max(R × Math.sqrt(value / reference), 2.5)
  Illy:  fill var(--primary-500), opacity 1
  comps: fill var(--grey-500), opacity 0.78
  data-brand attribute, cursor pointer
  on click: broadcastSelection(brand)
```

**Reference label**
```
text "○ = {reference}{unit}", x=containerWidth-2, y=13, text-anchor end
8px Tableau Light, color var(--grey-700)
```

**Value label — two cases based on r_inner**
```
case A — r_inner ≥ 10px (label inside inner circle):
  text formatted value, Baskerville italic
  font-size Math.min(12, r_inner × 0.38)
  fill var(--grey-900), text-anchor middle, y=cy+4

case B — r_inner < 10px (leader line + label above outer circle):
  line: x1=cx, y1=cy-r_inner-2, x2=cx, y2=cy-R-10
        stroke brand colour, stroke-width 0.8px
        stroke-dasharray 2 2, opacity 0.65
  text: x=cx, y=cy-R-13, text-anchor middle
        9.5px Baskerville italic, brand colour, data-brand attribute
```

**Brand label (below outer circle)**
```
x=cx, y=cy+R+16, text-anchor middle
10px (Illy) / 9px (comps) Tableau Light uppercase, brand colour
```

**Reference values by metric**
```
sov, bss → reference = 100,  unit = '%'
eqr, ebl → reference = 1.0,  display value × 100 as '%'
nps      → reference = 100,  unit = ' people'
```

**Enter:** inner circles r from 0, 400ms d3.easeBackOut, stagger 40ms per brand.
Outer circles render immediately.

```js
// Data shapes:
// sov/bss: [{ brand:'Illy', value:0.26 }, { brand:'Starbucks', value:80.66 }, ...]
// eqr/ebl: [{ brand:'Illy', value:0.082 }, ...]  reference=1.0, display ×100
// nps:     [{ brand:'Illy', value:45 }, ...]      reference=100
```

---

### Chart 12 · Progress ring
**Metrics:** `eqr` (alt) · `nps` (alt) · `sov` (alt)

Full circle ring. Track = dim 360° ring. Fill arc = clockwise from 12 o'clock.

**Layout**
```
sort:    Illy → slot 0. Others sorted desc by value.
slotW:   containerWidth / nBrands
R:       Math.min(52, slotW/2 - 10)
sw:      Math.max(8, R × 0.22)          (stroke width)
cx[i]:   slotW × i + slotW / 2
cy:      R + 14
SVG height: R×2 + 50
```

**Marks**
```
track arc (context — renders immediately, no transition):
  d3.arc().innerRadius(R-sw).outerRadius(R)
    .startAngle(-Math.PI).endAngle(Math.PI)
  fill var(--grey-700), opacity 0.30

fill arc (data — animated):
  startAngle = -Math.PI/2   (12 o'clock)
  endAngle   = -Math.PI/2 + (value/reference) × 2π
  cornerRadius = sw / 2
  d3.arc().innerRadius(R-sw).outerRadius(R)
    .startAngle(-π/2).endAngle(endAngle).cornerRadius(sw/2)
  Illy:  fill var(--primary-500), opacity 1
  comps: fill var(--grey-500), opacity 0.80
  data-brand attribute, cursor pointer, on click: broadcastSelection(brand)
```

**Labels**
```
value (centred in ring):
  Baskerville italic, font-size Math.min(13, R × 0.28)
  brand colour, text-anchor middle, x=cx, y=cy+4
  display: ×100 for ratio metrics, integer for count metrics

brand (below ring):
  x=cx, y=cy+R+16, text-anchor middle
  10px (Illy) / 9px (comps) Tableau Light uppercase, brand colour
  data-brand attribute
```

**Reference values:** same as Chart 11.

**Enter:** fill arc endAngle animates 0→final via attrTween, 700ms d3.easeCircleOut.
Track renders immediately.
**Compact:** sw cap = Math.max(5, R × 0.20).

---

### Chart 13 · Waffle chart
**Metrics:** `nps` (primary) · `eqr` (alt)

100-dot grid per brand.

**Geometry**
```
dot radius:  3px  (diameter 6px)
step:        8px  (dot diameter 6px + gap 2px)
cols, rows:  10 × 10
blockW:      10 × 8 - 2 = 78px
blockH:      10 × 8 - 2 = 78px
SVG height:  blockH + 22 = 100px

brand gap:   (containerWidth - nBrands × 78) / (nBrands - 1)
             minimum 10px; if below minimum, reduce step to 7px (blockW/H = 68px)

Illy: leftmost block. Others sorted desc by value.
blockX[i] = i × (blockW + brandGap)
```

**Dot colours**

For `eqr` and single-metric waffles:
```
i < nFilled:
  Illy:  fill var(--primary-500), opacity 1
  comps: fill var(--grey-500), opacity 0.78
i ≥ nFilled:
  all:   fill var(--grey-700), opacity 0.50
```

For `nps` (three-state):
```
i < nPromoters:
  Illy:  fill var(--primary-500), opacity 1
  comps: fill var(--grey-500), opacity 0.78
nPromoters ≤ i < nPromoters + nPassives:
  all:   fill var(--grey-700), opacity 0.50
i ≥ nPromoters + nPassives:
  all:   fill var(--primary-900), opacity 0.85
```

Fill order: left→right, top→bottom. `i = row×10 + col`.

**Calculations**
```
nFilled    = Math.round((value / reference) × 100)
nPromoters = Math.round(promoterFraction × 100)
nPassives  = Math.round(passiveFraction × 100)
```

**Brand label**
```
x = blockX + blockW/2, y = blockH + 16, text-anchor middle
10px (Illy) / 9px (comps) Tableau Light uppercase, brand colour
data-brand attribute
```

**Enter:** fade entire per-brand block opacity 0→1, 500ms d3.easeCubicOut,
stagger 80ms per brand. (100 individual tweens is too expensive.)
**Compact:** reduce to 8×8 grid (64 dots), step 7px, blockW/H 54px, if height < 160px.

**Reference values:** `nps` = 100 (promoters out of 100 people). `eqr` = 1.0 (× 100 for nFilled).

```js
// nps data shape
[{ brand:'Illy', promoterFraction:0.45, passiveFraction:0.30, detractorFraction:0.25 }, ...]
// eqr data shape
[{ brand:'Illy', value:0.082 }, ...]  // reference=1.0
```

---

### Chart 14 · Horizontal bar
**Metrics:** `cra` (primary) · `sstsr` (primary) · `dvtr` (alt)

**Dimensions**
```
margin: top 12, right 52, bottom 8, left 88
        88px left margin reserved for brand labels
        52px right margin for value labels
```

**Layout**
```
sort: brands by value desc; Illy forced to top row regardless of rank
scaleBand: padding 0.25, bandwidth capped at 32px
barH = Math.min(bandwidth, 20px)
barY = rowY + (bandwidth - barH) / 2   (vertically centred)

Illy separator: 0.5px var(--grey-700) horizontal rule
  y = illyRowY + bandwidth + band.step() × 0.13
  x1 = -margin.left, x2 = iW + margin.right
```

**Marks**
```
bars:
  Illy:  fill var(--primary-500), rx=2
  comps: fill var(--grey-500), rx=2
  data-brand attribute, cursor pointer

brand labels (left of bars):
  x=-8, y=rowMidY+4, text-anchor end
  10px (Illy) / 9px (comps) Tableau Light uppercase
  fill var(--primary-500) Illy / var(--grey-100) comps

value labels (right of bars):
  x = scale(value) + 8, y = rowMidY + 4, text-anchor start
  11px Baskerville italic, brand colour
  format: value + unit string (e.g. '5.95%')

benchmark rule (if config.benchmark provided):
  x = scale(config.benchmark), y1=0, y2=totalBandHeight
  stroke var(--grey-300), stroke-width 1px, stroke-dasharray 3 3
  label "BENCHMARK": x=scale(config.benchmark), y=-5, text-anchor middle
                     8.5px Tableau Light, color var(--grey-300)
```

**Axes**
```
x-axis: d3.axisBottom, tickSize = -(nBrands × maxRowH)
        gridlines: var(--grey-700) opacity 0.35, stroke-dasharray 2 3
        9px Tableau Light var(--grey-500), no domain line
        tick format: d => d + unit  (e.g. d => d + '%')

no y-axis (brand labels serve this role)
```

**Units and domains per metric**
```
cra:   unit='%',  benchmark=config (industry avg if available)
sstsr: unit='',   domain [0, max×1.1], benchmark=~0.2
dvtr:  unit='',   domain [0, 5],       benchmark=~3.2
```

**Enter:** bars grow from x=0 to full width, 600ms d3.easeCubicOut, stagger 50ms top→bottom.
Value labels fade in 300ms after bars complete.

```js
[{ brand:'Illy', value:0.66 }, { brand:'Nespresso', value:5.95 }, ...]
```

---

### Chart 15 · Vertical bar stacked
**Metrics:** `eqr` (primary) · `ebl` (primary)

**Dimensions**
```
margin: top 10, right 8, bottom 26, left 30
```

**Rendering order (important)**
```
For each brand column (sorted: comps first, Illy last):
  1. passive rect: y=0, height=yScale(active)           ← visually TOP portion
  2. active rect:  y=yScale(active), height=iH-yScale(active) ← visually BOTTOM portion
  3. separator line: y=yScale(active), x1=barX, x2=barX+barW
```

**Marks**
```
Illy active:   fill var(--primary-500)
Illy passive:  fill var(--primary-700)   #E0697D
comp active:   fill var(--grey-500)
comp passive:  fill var(--grey-700)
separator:     stroke var(--grey-900), stroke-width 2px, pointer-events none

scaleBand padding 0.28, all brands, no forced Illy position
data-brand on active rect (the meaningful segment)
cursor pointer, on click: broadcastSelection(brand)
```

**Axes**
```
y-axis: scaleLinear [0,1], display 0–100%
        d3.axisLeft, 4 ticks, tickSize -iW
        gridlines var(--grey-700) opacity 0.35, stroke-dasharray 2 3
        tick format: d => Math.round(d×100)+'%'
        9px Tableau Light var(--grey-500), remove domain line

x-axis: brand name labels, no domain line, tickSize 0, tickPadding 6
        9px Tableau Light, var(--primary-500) Illy / var(--grey-100) comps
```

**Enter:** bars grow from y=iH (baseline), 600ms d3.easeCubicOut.
Illy enters 80ms after all competitor bars.

**Compact:** omit y-axis. Show % value atop active segment:
11px Baskerville italic, brand colour, y=yScale(active)-4.

```js
// eqr: active = quality / total, passive = 1 - active
// ebl: active = beyondLikes / total, passive = 1 - active
[{ brand:'Illy', active:0.082, passive:0.918 }, ...]
```

---

### Chart 16 · Scale · Figma
**Metrics:** `sop` (primary) · `bt` (primary) · `dvtr` (alt)

Matches Figma Image 4. Brand name above bar. Score to the right. Solid fills, no gradients.

**Dimensions per brand row**
```
trackW:     containerWidth - 52px   (52px reserved for score column)
trackH:     34px
barH:       28px  (3px padding top and bottom within track)
barY:       (trackH - barH) / 2 = 3px
rowGap:     5px between brand rows
axisH:      18px below all rows
```

**Marks (per brand row, rendered as div + SVG hybrid)**
```
brand name div:
  font-family Tableau Light, font-size 8.5px (Illy) / 8px (comps)
  text-transform uppercase, letter-spacing 0.10em
  color var(--primary-500) Illy / var(--grey-500) comps
  margin-bottom 3px

track SVG (trackW × trackH):
  background rect: x=0 y=0 w=trackW h=trackH fill var(--grey-900) rx=4

  filled bar:
    x=0, y=3, w=xScale(value), h=28, rx=4
    Illy: fill var(--primary-500)
    comps: fill var(--grey-500)
    NO gradient, solid fill only
    data-brand attribute

  benchmark marker (if within domain):
    x=xScale(benchmark), y=-2, height=38  (overflows track ±2px)
    stroke var(--grey-100), stroke-width 2.5px

score div (52px wide):
  font-family Baskerville, font-style italic, font-size 15px (Illy) / 14px (comps)
  padding-left 10px, text-align right (or start), brand colour

Illy separator: 0.5px var(--grey-700) rule below Illy's score div
```

**Sort:** highest value at top. Illy rendered first regardless of rank.

**Axis SVG (trackW × 18px, below all rows)**
```
d3.axisBottom(xScale).ticks(5).tickSize(0)
8px Tableau Light, color var(--grey-700), no domain line
```

**Domains and benchmarks**
```
sop:  xDomain [2.5, 3.5], benchmark 3.0
bt:   xDomain [2.7, 3.2], benchmark 3.5
dvtr: xDomain [0, 5],     benchmark 3.2
```

**Enter:** bars grow from w=0, 600ms d3.easeCubicOut, stagger 60ms per row (top→bottom).
Benchmark line fades in 200ms after bars.

**Compact:** trackH 24px, barH 18px, brand-name font 7.5px, score font 12px.

```js
[{ brand:'Illy', value:2.72 }, { brand:'Nespresso', value:3.30 }, ...]
```

---

### Chart 17 · Multi-scale dots
**Metrics:** `ba` (primary)

Run `guardBaAttributes(data)` first. If false: render centred placeholder text
"Attribute data unavailable — pipeline error", 11px Tableau Light, var(--grey-700).

**Dimensions**
```
margin: top 30 (column headers), right 12, bottom 12, left 88
ROW:    28px per brand row
SVG height: nBrands × ROW + margin.top + margin.bottom
```

**Sort:** Illy first. Others sorted by mean of all attributes desc.

**Marks (per brand row)**
```
cy = margin.top + brandIndex × ROW + ROW/2

track line (context):
  x1=xScale(min attr value), x2=xScale(max attr value), y1=cy, y2=cy
  stroke var(--grey-700), stroke-width 0.5px, opacity 0.6

connector / range line:
  x1=xScale(min attr), x2=xScale(max attr), y1=cy, y2=cy
  stroke brand colour, stroke-width 1px, opacity 0.20

attribute dots (5 per row — Trust, Innovation, Value, Style, Heritage):
  cx = xScale(brand[attributeName]), cy = cy
  Illy:  r=6px, fill var(--primary-500), stroke var(--grey-800) 1.5px
  comps: r=5px, fill var(--grey-500), opacity 0.85, stroke var(--grey-800) 1.5px
  data-brand attribute, cursor pointer, on click: broadcastSelection(brand)

brand label:
  x = -8, y = cy+4, text-anchor end
  10px (Illy) / 9px (comps) Tableau Light uppercase, brand colour
  data-brand attribute
```

**Column headers (above Illy row only)**
```
Attribute names abbreviated to 3 chars, uppercase
x = xScale(illy[attr]), y = margin.top - 10, text-anchor middle
7.5px Tableau Light, color var(--grey-700)
(Position over Illy's actual dot value — not evenly spaced)
```

**X-axis (below last row)**
```
d3.axisBottom, domain from config (default [2.5, 5])
tickSize = -(nBrands × ROW + 4)
gridlines: var(--grey-700) opacity 0.30
9px Tableau Light, color var(--grey-700)
```

**Enter:** per row, dots scale from r=0, 400ms d3.easeBackOut.
Rows stagger 60ms. Dots within each row stagger 20ms.

```js
[{ brand:'Illy', Trust:3.8, Innovation:3.2, Value:2.9, Style:4.1, Heritage:4.4 }, ...]
```

---

### Chart 18 · Dot matrix
**Metrics:** `nps` (alt to waffle)

**Geometry**
```
dot radius:  2.5px  (diameter 5px)
step:        7px    (5px + 2px gap)
cols, rows:  10 × 10 per brand
blockW:      10 × 7 - 2 = 68px
blockH:      10 × 7 - 2 = 68px
SVG height:  blockH + 22 = 90px

Illy: leftmost. Others sorted desc by promoterFraction.
brandGap = (containerWidth - 5 × 68) / 4
minimum brandGap = 8px; if below, reduce step to 6px (blockW/H = 58px)
blockX[i] = i × (blockW + brandGap)
dots start at y=16 (after brand label)
```

**Brand label (above block)**
```
x = blockX + blockW/2, y=12, text-anchor middle
9.5px (Illy) / 8.5px (comps) Tableau Light uppercase, brand colour
```

**Dot colours by index i (0..99)**
```
nPromoters = Math.round(promoterFraction × 100)
nPassives  = Math.round(passiveFraction  × 100)
dot position: col = i%10, row = Math.floor(i/10)
              cx = blockX + col×step + r, cy = 16 + row×step + r

i < nPromoters:
  Illy:  fill var(--primary-500), opacity 1
  comps: fill var(--grey-500), opacity 0.78

nPromoters ≤ i < nPromoters+nPassives:
  all:   fill var(--grey-700), opacity 0.50

i ≥ nPromoters+nPassives:
  all:   fill var(--primary-900), opacity 0.85
```

**Enter:** fade per-brand block opacity 0→1, 400ms d3.easeCubicOut, stagger 80ms.

```js
[{ brand:'Illy', promoterFraction:0.45, passiveFraction:0.30, detractorFraction:0.25 }, ...]
```

---

### Chart 19 · Small multiples
**Metrics:** `bss` (context) · `sov` (context) · `svt` (context)

Five mini panels in one row. Same scale across all panels.

**Dimensions**
```
panelW:    Math.floor(containerWidth / 5)   (no gap — tight grid)
cellH:     88px
labelH:    18px
SVG h:     cellH + labelH = 106px

inner width per panel:  panelW - 8   (4px padding each side)
inner height per panel: cellH - 4    (2px padding top and bottom)
innerX origin:          panelX + 4
innerY origin:          2
```

**Y scale (shared across all 5 panels)**
```
globalMin = d3.min of all brands' values across all time periods
globalMax = d3.max of all brands' values across all time periods
do NOT normalise per-brand — defeats the purpose of comparison
```

**X scale (shared across all 5 panels)**
```
d3.scalePoint, domain: all time period labels
range: [0, inner panel width]
```

**Marks per panel**
```
background rect:
  x=panelX+4, y=2, w=panelW-8, h=cellH-4
  fill var(--grey-900), rx=3

area fill:
  d3.area().x(d=>x(d.month)).y0(innerH).y1(d=>y(d[brand]))
  .curve(d3.curveMonotoneX)
  Illy: fill var(--primary-500), fill-opacity 0.50
  comps: fill brand var(--grey-500), fill-opacity 0.28

line:
  d3.line().x(d=>x(d.month)).y(d=>y(d[brand]))
  .curve(d3.curveMonotoneX)
  Illy: stroke var(--primary-500), stroke-width 1.5px
  comps: stroke var(--grey-500), stroke-width 1px
  data-brand attribute, cursor pointer

no axes, no tick labels, no gridlines inside panels
```

**Brand label (below each panel)**
```
x = panelX + panelW/2, y = cellH+14, text-anchor middle
9.5px (Illy) / 8.5px (comps) Tableau Light uppercase, brand colour
data-brand attribute
```

**Enter:** lines draw left→right, 600ms d3.easeLinear. Areas fade in simultaneously.
**Compact:** cellH=52px, total SVG height=70px.

Same data shape as Chart 06 (wide format, one column per brand).

---

### Chart 20 · Arc
**Metrics:** `nps` (Illy callout)

**HOLD: run `guardNpsRecalculated(data)` before rendering.**
If guard returns false: render empty state with "NPS recalculation pending",
11px Tableau Light, var(--grey-700), centred. Do NOT deploy this instance.

**Geometry**
```
cx:          containerWidth × 0.40
cy:          containerHeight × 0.85
R:           Math.min(cx, cy × 0.70)
sw:          Math.max(20, R × 0.18)   (stroke width)

semicircle:  startAngle -π/2 (9 o'clock) → endAngle +π/2 (3 o'clock)
             sweeps through 12 o'clock (top)
```

**Marks**
```
track arc (full 360° — context):
  d3.arc().innerRadius(R-sw).outerRadius(R)
    .startAngle(-Math.PI).endAngle(Math.PI)
  fill var(--grey-700), opacity 0.28

fill arc (Illy value):
  startAngle = -π/2
  endAngle   = -π/2 + ((value + 100) / 200) × π
               maps NPS range [-100, +100] onto semicircle [0, π]
  cornerRadius = sw / 2
  d3.arc().innerRadius(R-sw).outerRadius(R)
    .startAngle(-π/2).endAngle(endAngle).cornerRadius(sw/2)
  fill var(--primary-500), solid, NO gradient
```

**Labels**
```
centre value:
  text: value (integer NPS score), Baskerville italic, 52px
  fill var(--primary-500), x=cx, y=cy-10, text-anchor middle

brand sub-label:
  text "ILLY COFFEE", 10px Tableau Light, letter-spacing 0.08em
  fill var(--grey-500), x=cx, y=cy+14, text-anchor middle

endpoint labels — use sin/cos (NOT d3.arc for endpoint positions):
  left:  x = cx + Math.sin(-π/2) × (R+12)  →  cx - R - 12
         y = cy - Math.cos(-π/2) × R        →  cy
         text "−100", text-anchor end, 9px Tableau Light, var(--grey-500)

  right: x = cx + Math.sin(+π/2) × (R+12)  →  cx + R + 12
         y = cy - Math.cos(+π/2) × R        →  cy
         text "+100", text-anchor start, 9px Tableau Light, var(--grey-500)
```

**Competitor list (right panel, hide if containerWidth < 400px)**
```
xOrigin:  containerWidth × 0.64
yOrigin:  cy - R × 0.65
rowGap:   22px
sort:     competitors by NPS desc

per row:
  circle: r=4px, fill var(--grey-500), cx=xOrigin, cy=yOrigin+i×22
  name:   x=xOrigin+12, y+4, 10px Tableau Light, var(--grey-300)
  score:  x=containerWidth-16, text-anchor end
          12px Baskerville italic, var(--grey-500)
```

**Enter:** fill arc endAngle animates from -π/2 to final value via attrTween,
800ms d3.easeCircleOut. Track renders immediately.

```js
// Requires true NPS scores — NOT Likert means (2.52–3.24)
{ illy: { nps: 24 },
  competitors: [
    { brand:'Peets', nps:18 }, { brand:'Lavazza', nps:14 },
    { brand:'Starbucks', nps:12 }, { brand:'Nespresso', nps:8 }
  ]
}
```

---

## 9 · Instance configurations

```js
// tam.js
initBAN('chart-tam', {
  value: null, prefix: '$', unit: 'B', period: '2025',
  label: 'TOTAL ADDRESSABLE MARKET', sourceWorksheet: 'TAM_Data',
});

// cagr.js
initBAN('chart-cagr', {
  value: null, unit: '%', directional: true,
  label: 'COMPOUND ANNUAL GROWTH RATE', sourceWorksheet: 'Market_Data',
});

// svt.js
initLine('chart-svt', {
  straight: true, yFormat: '+.0%', unit: 'MoM % change',
  sourceWorksheet: 'Search_Volume', guard: 'svtZeros',
});

// sop.js
initScaleFigma('chart-sop', {
  domain: [2.5, 3.5], benchmark: 3.0,
  label: 'SHARE OF PREFERENCE',
  sourceWorksheet: 'Survey_Data', guard: 'likertNotPercent',
});

// cra.js
initHBar('chart-cra', {
  unit: '%', benchmark: null,
  label: 'CONVERSION RATE ANALYSIS', sourceWorksheet: 'Conversion_Data',
});

// bss.js — two instances
initArea100('chart-bss-trend', {
  label: 'BRANDED SEARCH SHARE · TREND',
  sourceWorksheet: 'Search_Share', brush: true,
});
initTreemapBar('chart-bss-snapshot', {
  label: 'BRANDED SEARCH SHARE · SNAPSHOT',
  sourceWorksheet: 'Search_Share', period: 'latest',
});

// dvtr.js
initBANs('chart-dvtr', {
  label: 'DISTINCTIVE VERBAL TONE RECOGNITION',
  sourceWorksheet: 'Survey_Data',
});

// sstsr.js
initHBar('chart-sstsr', {
  unit: '', benchmark: 0.2,
  label: 'SCROLL-STOP / THUMB-STOP RATIO', sourceWorksheet: 'Social_Data',
});

// eqr.js
initVBarStacked('chart-eqr', {
  activeKey: 'activeEngagements', passiveKey: 'passiveEngagements',
  label: 'ENGAGEMENT QUALITY RATIO', sourceWorksheet: 'Engagement_Data',
});

// sov.js — two instances
initArea100('chart-sov-trend', {
  label: 'SHARE OF VOICE · TREND', sourceWorksheet: 'Social_Share',
});
initTreemapBar('chart-sov-snapshot', {
  label: 'SHARE OF VOICE · SNAPSHOT',
  sourceWorksheet: 'Social_Share', period: 'latest',
});

// vom.js
initLine('chart-vom', {
  straight: false, yFormat: '+.0%', unit: 'MoM velocity',
  sourceWorksheet: 'Mentions_Data',
});

// nps.js
initWaffle('chart-nps', {
  reference: 100, valueKey: 'promoterCount',
  label: 'PROMOTERS · PER 100 RESPONDENTS',
  sourceWorksheet: 'Survey_Data', guard: 'npsRecalculated',
});
// Arc: built but NOT deployed — uncomment when NPS data is confirmed:
// initArc('chart-nps-callout', { guard: 'npsRecalculated', sourceWorksheet: 'Survey_Data' });

// ba.js
initMultiScale('chart-ba', {
  attributes: ['Trust','Innovation','Value','Style','Heritage'],
  domain: [2.5, 5], label: 'BRAND AFFINITY',
  sourceWorksheet: 'Affinity_Data', guard: 'baAttributes',
});

// ebl.js
initVBarStacked('chart-ebl', {
  activeKey: 'beyondLikes', passiveKey: 'likesOnly',
  label: 'BRAND ENGAGEMENT BEYOND LIKES', sourceWorksheet: 'Engagement_Data',
});

// bt.js
initScaleFigma('chart-bt', {
  domain: [2.7, 3.2], benchmark: 3.5, label: 'BRAND TRUST',
  sourceWorksheet: 'Survey_Data', guard: 'starbucksPresent',
});
```

---

## 10 · Known data issues

| Metric | Issue | Guard function | Action |
|---|---|---|---|
| `nps` | Values are Likert means 2.52–3.24, not NPS scores | `guardNpsRecalculated` | Use waffle + BANs; block arc |
| `ba` | 4 of 5 brands identical at 4.060 — pipeline error | `guardBaAttributes` | Render placeholder, no chart |
| `svt` | Q4 values all zero — pipeline unpopulated | `guardSvtZeros` | Render with ⚠ annotation |
| `bt` | Starbucks missing from dataset | `guardStarbucksPresent` | Console warn, render remaining brands |
| `sop` `dvtr` `bt` | Likert means served instead of percentages | `guardLikertNotPercent` | Console warn, skip render |

---

## 11 · Deliverables checklist

- [ ] `tokens.css` — all CSS custom properties
- [ ] `lib/scaffold.js`
- [ ] `lib/selection.js`
- [ ] `lib/tooltip.js`
- [ ] `lib/data-guards.js` — all 5 guard functions
- [ ] 20 chart modules in `charts/`
- [ ] 17 instance configs (15 metrics, bss + sov have 2 instances each)
- [ ] All charts tested at 200×600px compact Tableau wide-ratio mode
- [ ] Selection propagates across all chart instances via event bus
- [ ] `arc.js` fully built; instance file comment-blocked pending NPS recalculation
- [ ] `multiscale.js` renders placeholder when `guardBaAttributes` fails
- [ ] `line.js` (svt instance) renders Q4 annotation when `guardSvtZeros` returns true
