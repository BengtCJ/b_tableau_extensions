# Spec: Metric ID filter in fetchAndRender
*BP Tableau Extensions · index.html*

## What and why

The shared data worksheet returns all 15 metrics in one query (one row per brand × indicator_id × quarter). Without filtering, `fetchAndRender` passes all rows to the chart and each brand gets a nonsense aggregated value. Each chart extension instance must filter to its own `indicator_id` before rendering.

## Changes required

### 1. DEFAULTS object

Add one key:

```js
metricId: '',
```

### 2. fetchAndRender — column discovery

After the existing `brandIdx` / `valueIdx` / `periodIdx` lookups, add:

```js
const indicatorIdx = cols.findIndex(c => c.fieldName === 'indicator_id');
```

No error if missing — guard is handled in the filter below.

### 3. fetchAndRender — data mapping and filter

Replace the existing `.map` + `.filter` block with:

```js
const data = rows.map(row => ({
  name:      row[brandIdx].formattedValue || row[brandIdx].value,
  value:     parseFloat(row[valueIdx].value) || 0,
  period:    periodIdx  !== -1 ? (row[periodIdx].formattedValue  || row[periodIdx].value)  : null,
  indicator: indicatorIdx !== -1 ? row[indicatorIdx].value : null,
}))
.filter(d => d.value > 0)
.filter(d => !CONFIG.metricId || d.indicator === CONFIG.metricId);
```

The `!CONFIG.metricId` guard means a blank metricId returns all rows — useful for inspecting raw sheet output during setup, and safe in production because no real chart config will have a blank metricId.

### 4. Settings dialog — new field

Add to the `fields-grid` div, after the Value Field field:

```html
<div class="field">
  <label>Metric ID</label>
  <input id="s-metricId" type="text" value="${CONFIG.metricId}"
    placeholder="e.g. sop · bss · vom">
  <div class="hint">Matches indicator_id column. Leave blank to return all.</div>
</div>
```

### 5. Settings save handler

Add alongside the other `CONFIG.*` assignments:

```js
CONFIG.metricId = document.getElementById('s-metricId').value.trim().toLowerCase();
```

`.toLowerCase()` guards against case mismatches between the settings input and the Tableau field value (Tableau sometimes returns indicator_id values in mixed case depending on the data source).

### 6. Settings load from Tableau Settings API

No change needed — the existing loop already reads all CONFIG keys from settings:
```js
Object.keys(DEFAULTS).forEach(k => {
  const val = settings.get(k);
  if (val) CONFIG[k] = val;
});
```
Adding `metricId` to DEFAULTS means it is automatically persisted and restored.

## Validation

- Open extension in browser (no Tableau). Settings dialog should show the Metric ID field.
- In Tableau, configure two instances of the extension on the same dashboard pointing at the same worksheet. Set one to `metricId: sop`, the other to `metricId: vom`. Confirm each renders only its own metric's data.
- Set metricId to a value that does not exist in the data (e.g. `zzz`). Chart should render empty — the existing empty-data path handles this; confirm no JS error is thrown.
- Leave metricId blank. Confirm all brands render with summed/mixed values (expected, not a bug — blank is a debug affordance).

## What not to change

- Field name `indicator_id` is hardcoded in the column lookup above. Do not make it configurable via settings — it is a fixed column name on the shared sheet and adding another config field increases settings dialog complexity for no benefit.
- Do not add `indicator` to the data object passed to chart render functions — charts do not need it and it would require touching every render function signature.
