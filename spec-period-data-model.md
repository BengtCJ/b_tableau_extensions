# Spec: Period data model — snapshot vs trend
*BP Tableau Extensions · index.html*

## How period data flows

The extension reads whatever column `CONFIG.periodField` names (default `'Quarter'`). Every row in the Tableau summary data gets a `period` property. No aggregation happens inside the extension — it relies on Tableau's summary to supply one row per brand × indicator × period.

## Snapshot vs trend chart types

After the `metricId` filter, `fetchAndRender` checks `CONFIG.chart` and either:

- **Reduces to the latest period** — takes `periods[periods.length - 1]` (last period in Tableau's returned order, which is chronological). One row per brand reaches the chart.
- **Passes all periods** — the chart owns its own x-axis and plots a series per brand.

| Behaviour | Chart types |
|---|---|
| **Snapshot** (auto-reduces to latest period) | `ban`, `bans`, `bar`, `hbar`, `donut`, `bubbles`, `treemap-bar`, `waffle`, `dot-matrix`, `arc`, `progress-ring`, `inset-bubble`, `vbar-stacked`, `scale-figma`, `multiscale` |
| **Trend** (all periods passed through) | `line`, `line-smooth`, `line-straight`, `area-100`, `small-multiples`, `stream`, `slope` |

The `multiPeriodCharts` constant in `fetchAndRender` is the authoritative list. If a new chart type is added that needs all periods, add it there.

## Switching between quarterly and monthly data

`periodField` names the **column**, not the value. To switch the time granularity:

| Data granularity | Setting |
|---|---|
| Quarterly (default) | `periodField = 'Quarter'` |
| Monthly | `periodField = 'Month'` (or whatever the column is named in the sheet) |

The Tableau sheet must have the relevant column on the Rows shelf (or Detail mark) for it to appear in `getSummaryDataAsync()`. The extension does not transform or reformat period values — they are used as-is for x-axis labels and period matching.

## Auto-latest-period behaviour (v2026-05-15.8)

For snapshot charts, `fetchAndRender` automatically reduces multi-period data to the last period in the dataset:

```js
const periods = [...new Set(data.map(d => d.period).filter(Boolean))];
if (periods.length > 1) {
  const latest = periods[periods.length - 1];
  data = data.filter(d => d.period === latest);
}
```

"Last" means last in Tableau's return order, which follows the sheet's sort order (normally chronological). This is not an alphabetical or date sort — it trusts Tableau's ordering.

## Pinning to a specific period (planned — see spec-period-selector.md)

`CONFIG.periodId` (not yet implemented in the UI) will let a dashboard author pin the extension to a named period (e.g. `'Q3 2024'`). When set it overrides the auto-latest behaviour for snapshot charts and acts as an explicit filter for trend charts. See `spec-period-selector.md` for the full implementation plan.

## What not to do

- Do not add period aggregation logic inside individual chart render functions — all period reduction happens in `fetchAndRender` before the router.
- Do not assume period strings are ISO dates — they are whatever Tableau formats them as (e.g. `'Q3 2024'`, `'September 2024'`, `'Sep'`).
- Do not sort periods alphabetically to find the latest — use positional order from the data.
