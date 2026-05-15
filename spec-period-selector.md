# Spec: Period selector in fetchAndRender
*BP Tableau Extensions · index.html*

## What and why

The shared worksheet returns data for multiple quarters in one query (one row per brand × indicator_id × quarter). Charts that need a single-period snapshot — BAN, bar, slope, scale — currently receive all quarters and either aggregate them or display the first/last arbitrarily. Each extension instance must be lockable to a specific period value (e.g. `Q3 2024`) so a dashboard can show different quarters side-by-side, or so a BAN tile always shows the correct quarter's number.

`periodField` already configures *which column* holds the period value. This spec adds `periodId` to configure *which period value* to display.

## Changes required

### 1. DEFAULTS object

Add one key after `periodField`:

```js
periodId: '',
```

Empty string means "pass all periods to the chart" — the existing behaviour.

### 2. fetchAndRender — period filter

After the existing `metricId` filter, add a second filter:

```js
if (CONFIG.periodId) data = data.filter(d => d.period === CONFIG.periodId);
```

The guard means a blank `periodId` is a no-op — line charts continue to show the full trend.

No column-discovery change needed: `period` is already mapped from `periodField` in the existing data extraction block.

### 3. Module-level period cache

Add one variable near the top of the script (alongside any other module-level state):

```js
let lastPeriods = [];
```

At the end of `fetchAndRender`, after the data array is built but **before** the period filter is applied, capture distinct period values:

```js
lastPeriods = [...new Set(rawData.map(d => d.period).filter(Boolean))];
```

This gives the settings dialog a live datalist populated from real data without requiring a separate fetch.

### 4. Settings dialog — new field

Add to the `fields-grid` div, immediately after the Period Field input:

```html
<div class="field">
  <label>Period / Quarter</label>
  <input id="s-periodId" type="text" value="${CONFIG.periodId}"
    placeholder="e.g. Q3 2024 — blank = all periods" list="s-period-ids">
  <datalist id="s-period-ids">
    ${lastPeriods.map(p => `<option value="${p}">`).join('')}
  </datalist>
  <div class="hint">Filters chart to one period. Leave blank for full trend.</div>
</div>
```

The datalist is populated from `lastPeriods` at dialog-render time. If no data has been fetched yet (extension just loaded, settings opened immediately), the list is empty but the free-text input still works.

### 5. Settings save handler

Add alongside the other `CONFIG.*` assignments:

```js
CONFIG.periodId = document.getElementById('s-periodId').value.trim();
```

No `.toLowerCase()` needed — period values from Tableau are case-sensitive strings (e.g. `"Q3 2024"`) and must match exactly.

### 6. Settings load from Tableau Settings API

No change needed — the existing loop already reads all CONFIG keys from settings:

```js
Object.keys(DEFAULTS).forEach(k => {
  const val = settings.get(k);
  if (val) CONFIG[k] = val;
});
```

Adding `periodId` to DEFAULTS means it is automatically persisted and restored.

## Validation

- Open extension in browser (`?chart=ban`). Settings dialog should show the Period / Quarter field below Period Field.
- Set `periodId` to `Q1` (matches sample data). BAN and bar charts should render only Q1 values.
- Set `periodId` to `''`. Line chart should show the full multi-period trend.
- Set `periodId` to `Q99` (not in data). Chart should render empty with no JS errors — the existing empty-data path handles this.
- In Tableau, open settings after at least one data fetch. The datalist should offer the quarters present in the sheet (e.g. `Q1 2024`, `Q2 2024`, …).
- Configure two dashboard tiles on the same worksheet, one with `periodId: Q2 2024` and one with `periodId: Q3 2024`. Confirm each renders only its own quarter.

## What not to change

- Do not expose `lastPeriods` outside the module — it is a UI convenience, not state.
- Do not add period-range or "latest N periods" logic — `periodId` is a single exact-match filter. Multi-period selection belongs in a separate spec if ever needed.
- Do not change the chart render functions — they already handle single-period data correctly by design (a bar chart with one period simply shows one bar per brand).
