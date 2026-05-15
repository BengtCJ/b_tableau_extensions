# plan.md — Architecture Blueprint

## Goal
A reusable library of custom D3 chart extensions for Tableau dashboards,
configurable via an in-extension settings dialog, hosted on GitHub Pages.

## Chart Library
| Chart | Status | Notes |
|-------|--------|-------|
| Donut | ✅ Built | BAN + legend + click; radius capped for wide containers |
| Bar | ✅ Built | Sorted; auto horizontal layout in landscape; 44px left margin |
| Line | ✅ Built | Multi-brand; click on paths/dots/labels; right-margin labels |
| BAN | ✅ Built | Headline number |
| BANs | ✅ Built | Multi-brand headline row |
| Line Straight / Smooth | ✅ Built | Shared `renderLineChart(straight)` |
| Slope | ✅ Built | Two-period comparison |
| Stacked 100% Area | ✅ Built | |
| Stream | ✅ Built | |
| Treemap / Treemap-bar | ✅ Built | |
| Bubbles / Inset bubble | ✅ Built | |
| Progress ring | ✅ Built | |
| Waffle | ✅ Built | |
| Horizontal bar | ✅ Built | |
| Vertical bar stacked | ✅ Built | |
| Scale Figma / Multiscale | ✅ Built | |
| Dot matrix | ✅ Built | |
| Small multiples | ✅ Built | |
| Arc (NPS callout) | ✅ Built | |

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

## Visual & Selection Quality (resolved 2026-05-15)
- **SVG sizing**: bar + line use `viewBox` + CSS `100%`; `getBoundingClientRect()` for dimensions — no iframe clipping
- **Bar layout**: auto-switches to horizontal when `H < 220` or `H < W * 0.35`
- **Donut radius**: capped at `min(H*0.44, available*0.5, 120)` with compact 60px fallback
- **ResizeObserver**: debounced 150ms to prevent re-render stutter during panel drag
- **Label truncation**: `truncLabel()` helper clips long brand names at 8 chars when container < 500px
- **Line selection**: 18px transparent hit-target paths, dot click, right-margin label click
- **Selection state**: `_selectedBrand` validated against fresh data on every `fetchAndRender`
- **Font**: `Baskervville` (Google Fonts) primary, `Baskerville` fallback

## Future: Multiple Dashboards
Each dashboard gets its own Extension object configured via the settings dialog.
Same index.html serves all charts — no duplicate files needed.
