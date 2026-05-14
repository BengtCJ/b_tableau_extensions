# plan.md — Architecture Blueprint

## Goal
A reusable library of custom D3 chart extensions for Tableau dashboards,
configurable via an in-extension settings dialog, hosted on GitHub Pages.

## Chart Library
| Chart | Status | Notes |
|-------|--------|-------|
| Donut | ✅ Built | BAN + legend + interactive |
| Bar | ✅ Built | Sorted, selected brand highlighted |
| Line | ✅ Built | Multi-brand, selected on top |
| BAN | ✅ Built | Headline number |
| Bubble | 🔲 Planned | Packed circles, size = value |
| Progress bars | 🔲 Planned | Horizontal bars like existing vis_bss |

## Config System
Settings stored in Tableau Extensions Settings API.
Persists inside the .twb workbook file.
Configured via ⚙ settings dialog — no URL params needed.

Key settings:
- chart, worksheet, brandField, valueField, periodField
- parameter (Select Brand)
- selectedColor, neutralColor, bgColor

## Data Flow
```
Tableau Parameter (Select Brand)
  → ParameterChanged event
    → fetchAndRender(worksheet)
      → getSummaryDataAsync()
        → renderChart(data, selectedBrand)
          → renderDonut/Bar/Line/BAN
```

## Design System
```
buildDesign() → {
  donut: { banValue, banLabel, legendLabel, legendValue, legendSelected }
  bar:   { valueLabel, valueLabelSel, axisLabel, axisLabelSel }
  line:  { axisX, axisY, gridColor }
  ban:   { value, label }
}
```
Each spec: `{ font, style, color }`

## Hosting
- GitHub Pages: https://bengtcj.github.io/b_tableau_extensions/
- Single URL allowlisted on Tableau Cloud
- One manifest.trex file (points to base URL)
- Settings dialog handles all per-chart config

## Future: Multiple Dashboards
Each dashboard gets its own Extension object configured via the settings dialog.
Same index.html serves all charts — no duplicate files needed.
