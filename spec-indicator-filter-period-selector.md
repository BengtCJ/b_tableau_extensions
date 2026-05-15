# Spec: Indicator filter + period selector
*BP Tableau Extensions · index.html*
*Supersedes: spec-metric-id-filter.md*

---

## Overview

Two related changes that must be implemented together:

1. **Indicator filter** — each chart instance filters the shared worksheet to its own `indicator_id` client-side after fetching.
2. **Period selector** — snapshot chart types get a button strip (All · Q1 · Q2 · Q3 · Q4) that filters the already-fetched data client-side. No additional Tableau query on period change.

Both operate on data already in memory after the initial `getSummaryDataAsync` call. Neither requires a new worksheet query on interaction.

---

## Sheet configuration (Tableau side — not code)

Two hidden 1px text table sheets. Both have Run ID as a context filter.

**Snapshot sheet** — Rows: Brand Name Upper, indicator_id. No Quarter dimension.
Tableau returns one aggregated row per brand per indicator (SUM across all quarters).
Used by: ban, bans, bar, donut, hbar, scale-figma, waffle.

**Trend sheet** — Rows: Brand Name Upper, indicator_id, Quarter.
Tableau returns one row per brand per indicator per quarter.
Used by: line, area100, vom.

The period selector only applies to snapshot chart instances. Trend charts always receive all quarters and render them as a time series — no period selector needed.

---

## State additions

Add two module-level variables alongside existing `_lastData`, `_lastSelected` etc.:

```js
let _allData        = null;  // full unfiltered fetch result, preserved for period re-filter
let _selectedPeriod = 'all'; // 'all' or a quarter string e.g. 'Q4 2024'
```

`_allData` is set once per `fetchAndRender` call and never mutated. `_selectedPeriod` is reset to `'all'` whenever `fetchAndRender` is called (i.e. on data refresh or settings save), so the chart always opens in the all-time view.

---

## DEFAULTS additions

```js
metricId: '',
```

No default needed for `_selectedPeriod` — it is runtime state, not persisted config.

---

## fetchAndRender changes

### Column discovery

After existing brandIdx / valueIdx / periodIdx lookups add:

```js
const indicatorIdx = cols.findIndex(c => c.fieldName === 'indicator_id');
```

### Data mapping

Replace the existing map/filter block with:

```js
_allData = rows.map(row => ({
  name:      row[brandIdx].formattedValue || row[brandIdx].value,
  value:     parseFloat(row[valueIdx].value) || 0,
  period:    periodIdx    !== -1 ? (row[periodIdx].formattedValue    || row[periodIdx].value)    : null,
  indicator: indicatorIdx !== -1 ? (row[indicatorIdx].formattedValue || row[indicatorIdx].value) : null,
}))
.filter(d => d.value > 0)
.filter(d => !CONFIG.metricId || d.indicator === CONFIG.metricId);

_selectedPeriod = 'all';
```

Note: `_allData` now holds the indicator-filtered data. The period filter is applied separately at render time (see below), not here.

### Hand-off to renderChart

Replace the existing `renderChart(data, ...)` call at the end of fetchAndRender with:

```js
renderWithPeriod();
```

---

## renderWithPeriod — new function

Add this function. It applies the period filter to `_allData` and calls `renderChart`.

```js
function renderWithPeriod() {
  if (!_allData) return;

  const data = _selectedPeriod === 'all'
    ? _allData
    : _allData.filter(d => d.period === _selectedPeriod);

  renderChart(data, _lastSelected || _allData[0]?.name, _dashboard);
}
```

Also call `renderWithPeriod()` in place of `renderChart(...)` inside the ResizeObserver callback:

```js
const resizeObserver = new ResizeObserver(() => {
  clearTimeout(_resizeTimer);
  _resizeTimer = setTimeout(renderWithPeriod, 150);
});
```

---

## renderChart changes

### Router

No change to the switch statement. The period filter has already been applied before renderChart is called — chart functions receive clean pre-filtered data and need no awareness of period selection.

### Period selector injection — new helper

Add this function. It is called at the top of each snapshot render function before drawing the chart.

```js
const SNAPSHOT_CHARTS = ['ban', 'bans', 'bar', 'donut', 'hbar', 'scale', 'waffle'];

function injectPeriodSelector(el, availableHeight) {
  // Only inject for snapshot chart types
  if (!SNAPSHOT_CHARTS.includes(CONFIG.chart)) return 0;
  // Hide entirely if container is too short
  if (availableHeight < 160) return 0;

  const periods = [...new Set((_allData || []).map(d => d.period).filter(Boolean))].sort();
  // If only one period or no periods, selector adds no value — omit it
  if (periods.length <= 1) return 0;

  const strip = document.createElement('div');
  strip.style.cssText = `
    display: flex;
    gap: 4px;
    padding: 0 4px 6px 4px;
    justify-content: flex-end;
    flex-shrink: 0;
  `;

  const buttons = ['all', ...periods];
  buttons.forEach(p => {
    const isSelected = _selectedPeriod === p;
    const hasData    = p === 'all' || (_allData || []).some(d => d.period === p && d.value > 0);
    const btn = document.createElement('button');
    btn.textContent  = p === 'all' ? 'All' : p;
    btn.style.cssText = `
      padding: 2px 8px;
      border-radius: 3px;
      border: 1px solid ${isSelected ? 'transparent' : 'var(--grey-700)'};
      background: ${isSelected ? 'var(--primary-500)' : 'var(--grey-800)'};
      color: ${isSelected ? 'var(--grey-900)' : hasData ? 'var(--grey-500)' : 'var(--grey-700)'};
      font-size: 9px;
      font-family: ${FONT_LIGHT};
      letter-spacing: 0.05em;
      text-transform: uppercase;
      cursor: ${hasData ? 'pointer' : 'default'};
      opacity: ${hasData ? 1 : 0.4};
    `;
    if (hasData) {
      btn.onclick = () => {
        _selectedPeriod = p;
        renderWithPeriod();
      };
    }
    strip.appendChild(btn);
  });

  el.appendChild(strip);
  return strip.offsetHeight || 28; // return height consumed so chart can subtract it
}
```

### Calling injectPeriodSelector in render functions

At the top of each snapshot render function, after reading container dimensions and before drawing anything, add:

```js
const selectorH = injectPeriodSelector(el, H);
const chartH    = H - selectorH;
// use chartH in place of H for all subsequent layout calculations
```

Affected render functions: `renderDonut`, `renderBar`, `renderBarHorizontal`, `renderBAN`.
Not affected: `renderLine` (trend chart — no selector).

For `renderBAN` specifically the selector is most useful — a BAN showing a single number benefits most from period context. Ensure the BAN value label and brand name are vertically centred within `chartH` not `H`.

---

## Settings dialog changes

### New field

Add to the fields-grid div, after the Value Field field:

```html
<div class="field">
  <label>Metric ID</label>
  <input id="s-metricId" type="text" value="${CONFIG.metricId}"
    placeholder="e.g. sop · bss · vom">
  <div class="hint">Matches indicator_id column. Leave blank to return all.</div>
</div>
```

### Save handler

Add alongside existing CONFIG assignments:

```js
CONFIG.metricId = document.getElementById('s-metricId').value.trim().toLowerCase();
```

On settings save, reset period selection:

```js
_selectedPeriod = 'all';
```

---

## Settings API persistence

No change needed. The existing loop reads all DEFAULTS keys automatically:

```js
Object.keys(DEFAULTS).forEach(k => {
  const val = settings.get(k);
  if (val) CONFIG[k] = val;
});
```

`_selectedPeriod` is not persisted — it resets to 'all' on every data load, which is the correct behaviour.

---

## Period sort order

Periods are sorted with `.sort()` which works correctly for strings formatted as `Q1 2024`, `Q2 2024` etc. (lexical sort matches chronological order for this format). If the Quarter field in Tableau uses a different format, verify sort order against actual data values before shipping. If sort is wrong, replace `.sort()` with a custom comparator.

---

## Validation

**Indicator filter**
- Configure two extension instances on the same dashboard pointing at the same worksheet. Set one to `metricId: sop`, the other to `metricId: vom`. Each should render only its own metric's brand values.
- Set metricId to a non-existent value. Chart renders empty with no JS error.
- Leave metricId blank. All indicators returned — mixed values per brand, expected debug behaviour.

**Period selector**
- On a snapshot chart with four quarters of data: four buttons + All appear. All is selected by default.
- Click Q3. Chart re-renders with Q3 data only. No new network/Tableau request fires (verify in browser network tab — no additional calls).
- Click All. Chart returns to summed view.
- A quarter with no data for the selected indicator renders as a dimmed button. Clicking it does nothing.
- At H < 160: selector is absent. Chart renders full height.
- On a trend chart (line): no selector appears regardless of height.
- Resize the container. Selected period persists across the resize re-render.
- Save new settings. Period resets to All.

---

## What not to change

- Field name `indicator_id` is hardcoded in the column lookup. Do not make it a config field.
- Do not add `indicator` or `period` to the data objects passed into chart render functions — charts receive clean pre-filtered arrays of `{ name, value, period }` as before.
- Do not add a period selector to trend charts. They own their time axis.
