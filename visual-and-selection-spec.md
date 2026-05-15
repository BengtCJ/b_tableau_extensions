# Visual & Selection Fixes — Spec for Claude Code
*BP Tableau Extensions · index.html*
*Status: ready for implementation / agent testing*

---

## Context

Single-file Tableau dashboard extension. All charts live in `index.html`. No build step. Charts render inside a Tableau iframe — the container dimensions are controlled entirely by Tableau and can be any aspect ratio. The four active chart types are: **Donut**, **Bar** (vertical columns), **Line** (multi-brand trend), **BAN** (headline number).

---

## 1. Visual issues

### 1.1 SVG clipping at container edges

**Problem.** Bar and Line charts set fixed pixel dimensions on the SVG element via D3's `.attr('width', W)` / `.attr('height', H)`. When Tableau resizes the panel below those pixel values, the SVG overflows and gets hard-clipped by the iframe boundary — content at the edges disappears with no scroll.

**Confirmed fix.**
- Read container dimensions with `el.getBoundingClientRect()` instead of `el.offsetWidth` / `el.offsetHeight`.
- Set SVG dimensions as CSS (`style('width','100%').style('height','100%').style('display','block')`) rather than attributes. All internal scale domains still use the numeric `W`/`H` values — only the SVG element declaration changes.
- Add `overflow: hidden` to the `#chart` CSS rule so any edge overflow is contained rather than escaping the iframe.
- Remove `min-height: 300px` from `#chart` — this forces the container taller than Tableau allows and pushes content out of frame.

### 1.2 Bar chart in short/wide containers

**Problem.** Vertical column bars in a landscape-ratio container (e.g. a Tableau tile set to ~150px tall and 600px wide) become very squat and unreadable. Brand labels on the x-axis collide or get cut.

**Proposed fix — automatic layout switch.**
When the container is short or wide (suggested threshold: `H < 220` or `H < W * 0.35`), switch to a horizontal bar layout: brand labels on the left, bars growing right, value labels at bar end. This is the natural layout for landscape tiles.

**Alternatives to test if the threshold feels wrong:**
- Option A: fixed threshold `H < 200` only, ignoring aspect ratio.
- Option B: always use horizontal bars and remove the vertical layout entirely. Simpler code, consistent behaviour — worth testing whether the vertical form is ever actually needed in Tableau context.
- Option C: add a manual toggle in the settings dialog ("Bar orientation: Auto / Horizontal / Vertical") so the dashboard author controls it.

**Left margin.** The current left margin of `16px` clips y-axis tick labels. Increase to at least `44px` regardless of layout mode.

### 1.3 Donut in short/wide containers

**Problem.** The donut SVG radius `R` was calculated as `min(H * 0.44, W * 0.28)`. In a wide container `W * 0.28` can be large, but the flex row layout (BAN block + circle + legend) doesn't have room for it — the circle overflows into or behind the legend.

**Proposed fix.**
Calculate available horizontal space for the circle explicitly: `availableForCircle = W - (BANBlockWidth + legendWidth + gaps)` (approximately `W - 290` with current layout). Cap R as `min(H * 0.44, availableForCircle * 0.5)`. Add a hard upper cap (e.g. `120px`) and a lower compact cap for short containers (e.g. `60px` when `H < 200`).

**Alternatives to test:**
- Option A: the above calculation with tweaked constants — the magic numbers (`290`, `120`, `60`) should be tuned against real Tableau tile sizes.
- Option B: constrain the donut SVG with `max-width` and `max-height` CSS on the SVG element instead of computing R analytically — simpler but less precise.
- Option C: for very short containers (`H < 180`), hide the donut SVG entirely and show only the BAN value + a compact legend table. The donut becomes decorative at small sizes anyway.

### 1.4 ResizeObserver firing rate

**Problem.** The ResizeObserver re-renders on every resize event. In Tableau Cloud, panel drag events fire continuously — this causes dozens of full re-renders per second during a resize, visibly stuttering.

**Confirmed fix.** Debounce with `setTimeout` / `clearTimeout`, `150ms` delay. Cancel any in-progress transitions before re-render (not currently applicable since transitions aren't used, but relevant once they are).

### 1.5 Long brand name truncation

**Problem.** Brand names like "Peet's Coffee" at 10–11 characters overflow x-axis labels in the vertical bar chart and right-margin labels in the line chart.

**Proposed approach.** Truncate at ~9 characters with an ellipsis when the container width is below a threshold (suggested `500px`). This is already specified in `CHARTS.md` (`truncateLabel` constant). The threshold and character limit should be tested against actual brand name lengths in the data.

**Alternatives:**
- Option A: rotate x-axis labels 45° instead of truncating — preserves readability but takes more vertical space.
- Option B: use D3's `textLength` / `lengthAdjust` attributes to compress text into available bandwidth before falling back to truncation.

---

## 2. Selection issues

Selection currently works as two separate mechanisms that should stay in sync:

1. **Visual highlight** — re-render with the clicked brand as `selectedBrand`, applying colour/opacity differences locally within the chart.
2. **Tableau mark selection** — call `worksheet.selectMarksByValueAsync` on other worksheets to propagate the selection across the dashboard.

### 2.1 Line chart — no click handlers

**Problem.** The line chart renders paths and dots but attaches no click events. Clicking anywhere on the chart does nothing. This makes selection entirely broken for this chart type.

**Proposed fix.**
- Attach click to each line `path` element. Because SVG path stroke has zero hit area at non-stroke pixels, add a second invisible `path` with the same `d` attribute, `stroke: transparent`, and a wide `stroke-width` (e.g. 18px) as a hit target. The visible line sits behind it with `pointer-events: none`.
- Attach click to all `circle` dot elements.
- Attach click to the brand name labels rendered in the right margin.
- Z-order: render non-selected brands first, selected brand last so it always sits on top in SVG paint order.

**Uncertainty.** The 18px hit target width is a guess — test with real mouse behaviour. May need to be wider on touch/tablet interfaces used with Tableau on Surface or iPad.

### 2.2 Selection state lost on re-render

**Problem.** `_selectedBrand` is only initialised once from the Tableau parameter (`Select Client Brand`). After that, it persists across re-renders via `_lastSelected`. However, if the worksheet data changes (e.g. a Tableau filter is applied), `fetchAndRender` is called again and `_selectedBrand` is not reset — which is correct — but it's also not re-validated against the new data. If the selected brand is filtered out, the chart renders with a `selectedBrand` value that no longer exists in the dataset.

**Proposed fix.**
After fetching fresh data, check whether `_selectedBrand` exists in the new dataset:
```js
if (_selectedBrand && !data.find(d => d.name === _selectedBrand)) {
  _selectedBrand = data[0]?.name || null;
}
```

**Alternative:** instead of defaulting to `data[0]`, re-read the Tableau parameter value and try to match again. This keeps the Tableau parameter as the source of truth.

### 2.3 Tableau mark selection — silent failures

**Problem.** `selectMarksByValueAsync` is called on all worksheets except the source, wrapped in `Promise.allSettled` so failures are swallowed. This is correct behaviour, but there is no feedback when selection does or doesn't propagate. In Tableau Cloud, mark selection can fail silently due to permissions or worksheet name mismatches.

**No proposed fix for now — flag as known.**
The visual selection (re-render) always works regardless of whether Tableau propagation succeeds. If propagation is confirmed broken in Cloud, options to investigate:
- Option A: use Tableau's `applyFilterAsync` on the source worksheet instead of `selectMarksByValueAsync` — different API surface, may have better Cloud support.
- Option B: use the Extensions Settings API to store selected brand and have other extensions read it — decouples selection from Tableau mark selection entirely.
- Option C: log the `allSettled` results to console so failures are visible during debugging.

### 2.4 BAN chart — no selection, intentional

The BAN chart displays a single value for the selected brand. It does not need click handling — the displayed value already reflects `selectedBrand`. Confirm this is the intended behaviour. If click-to-select is wanted (e.g. clicking the BAN to cycle through brands), that would require a different interaction model.

### 2.5 Donut — selection highlight on legend

**Problem** (minor). When a brand is selected by clicking a pie segment, the BAN value and legend colour update correctly. However, clicking a legend item also correctly selects the brand. The hover state on pie segments (arc expand) is visual only and does not persist — this is correct.

**No fix needed.** Documenting as working as intended.

---

## 3. What is not yet addressed

- **Which chart types to park** — will be specified separately. The dropdown currently uses `<optgroup>` to separate Active from Coming Soon; parking logic will map onto that structure.
- **Tooltip component** — specified in `CHARTS.md` (`src/lib/tooltip.js`) but not yet implemented. Not blocking for current chart types.
- **Tableau parameter change → re-render** — the parameter event listener exists for the source worksheet but has not been tested end-to-end on Cloud after the Settings API refactor. Verify this is still wired correctly.
- **Font fallback** — `context.md` notes that `'Baskerville'` is a macOS system font and fails on Windows/Cloud. Switch to `'Baskervville'` (Google Fonts, double-v) with a `<link>` import. Two places: CSS line ~68 and `FONT_TITLE` constant. Low risk change, high impact on Cloud rendering.

---

## Testing notes for agents

- Test each chart type at three container sizes: `300×200` (compact landscape), `400×400` (square), `600×300` (wide landscape).
- Selection test: click a non-selected brand → confirm visual highlight switches AND `_selectedBrand` updates AND (if in Tableau) other worksheets update.
- Resize test: drag panel smaller → confirm no clipping, no stutter, chart re-renders after 150ms pause.
- Browser test: Chrome (primary Tableau Cloud browser), then Edge. Safari only if time allows.
- The extension falls back to sample data when `tableau` is not defined — all visual tests can be done in a plain browser tab without Tableau.
