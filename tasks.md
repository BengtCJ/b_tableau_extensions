# tasks.md — Task Checklist

## Done ✅
- [x] GitHub Pages hosting set up
- [x] manifest.trex created and loading in Tableau Desktop
- [x] Extension allowlisted on Tableau Cloud
- [x] Settings dialog with 2-column layout
- [x] All 4 chart types built (donut, bar, line, BAN)
- [x] Design system (buildDesign) with per-element font/colour
- [x] Brand names Tableau Light + uppercase throughout
- [x] Transparent background default
- [x] Background colour picker with transparent checkbox
- [x] ResizeObserver for responsive sizing
- [x] Dynamic Tableau API loading (no blocking)
- [x] Fallback to sample data outside Tableau
- [x] Settings save via Tableau Extensions Settings API
- [x] Parameter change listener (re-renders on brand select)
- [x] Filter change listener
- [x] Client brand highlights pink on load (parameter alias fix — prefer `value` over `formattedValue`)

## In Progress 🔄
- [ ] Verify settings persistence on Tableau Cloud after publish
- [ ] Connect to real worksheet data (vis_bss or similar)
- [ ] Confirm parameter change triggers re-render on Cloud

## Up Next 🔲
- [ ] Test all 4 chart types with real data
- [ ] Fix donut hole colour for non-transparent backgrounds
- [ ] Add bubble chart type
- [ ] Add progress bar chart type (matching vis_bss style)
- [ ] Create CLAUDE.md global file (~/.claude/CLAUDE.md)
- [ ] Document per-dashboard setup instructions for team

## Backlog 📋
- [ ] Tooltip on hover for all charts
- [ ] Animation on brand select change
- [ ] Export/screenshot button
- [ ] Dark/light mode toggle
- [ ] Multiple metrics comparison view
