# context.md — Live Session State

## Current State
Extension is live and working on Tableau Cloud.
Single file: `index.html` hosted on GitHub Pages.

## What's Working
- All 4 chart types render with sample data in browser
- Settings dialog opens on first load (no worksheet configured)
- Settings save via Tableau Extensions Settings API
- ResizeObserver re-renders on container resize
- Donut: BAN number + legend + interactive segment click
- Bar: sorted, selected brand highlighted, value labels
- Line: multi-brand, selected on top with dots
- BAN: headline number + brand name
- Brand names always Tableau Light + uppercase throughout
- Background: transparent by default (works on Cloud), colour picker for Desktop

## Known Issues / In Progress
- Settings persistence on Cloud needs verification after allowlisting with full data access
- Desktop background shows white when transparent set — workaround: set bgColor in settings to match dashboard
- Donut hole colour needs to match background — handled via fill-opacity:0 when transparent

## Recent Decisions
- Moved from URL params to Tableau Settings API for config (allowlist URL matching issue)
- Transparent background default — SVG dashboard background means colour matching unreliable
- Single HTML file architecture — no build step, no bundler
- Design system via `buildDesign()` function — per-element font/colour specs

## Tableau Setup
- Workbook: BP Diagnostic Tool (v2.1)
- Cloud site: [your org].online.tableau.com
- Extension allowlisted at: https://bengtcj.github.io/b_tableau_extensions/index.html
- GitHub repo: https://github.com/BengtCJ/b_tableau_extensions

## Next Up
- Connect extension to real Tableau worksheet data
- Verify settings persistence after publish to Cloud
- Test parameter change triggers re-render on Cloud
