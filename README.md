# BP Chart Extensions

Custom D3 chart extensions for the BP Diagnostic Tool in Tableau.

## Charts Available

| Chart | `chart=` value | Description |
|-------|---------------|-------------|
| Donut | `donut` | Donut chart with BAN + legend, selected brand highlighted |
| Bar | `bar` | Sorted bar chart with selected brand highlighted |
| Line | `line` | Multi-brand line chart, selected brand emphasised |
| BAN | `ban` | Single big headline number for selected brand |

---

## URL Parameters

All config is passed via URL parameters on the extension source URL.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `chart` | `donut` | Chart type: `donut`, `bar`, `line`, `ban` |
| `worksheet` | *(required)* | Exact name of the Tableau worksheet |
| `brandField` | `Brand Name Upper` | Field name for brand/category |
| `valueField` | `Raw Value` | Field name for the numeric measure |
| `periodField` | `Quarter` | Field name for time period (line chart only) |
| `indicatorField` | `indicator_name` | Field name for indicator filter |
| `indicatorValue` | *(empty)* | Value to filter indicator by |
| `parameter` | `Select Brand` | Name of the Tableau Select Brand parameter |
| `selectedColor` | `#E8527A` | Hex colour for the selected/highlighted brand |
| `neutralColor` | `#5a5a5a` | Hex colour for non-selected brands |
| `bgColor` | `transparent` | Background colour (match your dashboard bg) |

---

## Example URLs

### Donut chart
```
https://bengtcj.github.io/bp-tableau-extensions/index.html?chart=donut&worksheet=vis_bss&indicatorValue=Branded+Search+Share
```

### Bar chart
```
https://bengtcj.github.io/bp-tableau-extensions/index.html?chart=bar&worksheet=vis_bar&indicatorValue=Engagement+Quality+Ratio
```

### Line chart (requires period field)
```
https://bengtcj.github.io/bp-tableau-extensions/index.html?chart=line&worksheet=vis_line&periodField=Quarter
```

### BAN number
```
https://bengtcj.github.io/bp-tableau-extensions/index.html?chart=ban&worksheet=vis_ban&valueField=Raw+Value
```

---

## How to Add to Tableau

1. In Tableau Desktop, open your dashboard
2. From the **Objects** panel drag **Extension** onto the dashboard
3. Click **"Access Local Extensions"** (during development) or **"Find Extensions"**
4. Browse to `manifest.trex` and open it
5. The extension will load and connect to your workbook data

### Configuring the URL

The manifest points to the base URL. To pass config, edit the manifest's `<url>` tag:

```xml
<source-location>
  <url>https://bengtcj.github.io/bp-tableau-extensions/index.html?chart=donut&amp;worksheet=vis_bss</url>
</source-location>
```

**Note:** Use `&amp;` instead of `&` in the XML manifest for multiple parameters.

Or create multiple `.trex` files — one per chart — each with a different URL.

---

## Development / Testing

Open `index.html` directly in a browser (without Tableau) to see it render with sample data. This lets you test styling changes without needing Tableau open.

---

## Multiple .trex Files (Recommended)

Create one `.trex` file per chart type for easy reuse in Tableau:

- `donut-bss.trex` → donut chart for Branded Search Share
- `bar-eqr.trex` → bar chart for Engagement Quality Ratio  
- `line-trends.trex` → line chart for trends
- `ban-score.trex` → BAN for headline score

Just duplicate `manifest.trex`, rename it, and change the `<url>` and `<name>` fields.
