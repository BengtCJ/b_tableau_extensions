# BP Tableau Extensions — CLAUDE.md

## Project
Custom D3 chart extensions for Tableau dashboards (BP Diagnostic Tool).
Single file: `index.html`. No build step. Deploy = `git push` to GitHub Pages.

## Structure
```
bp-tableau-extensions/
  index.html       ← entire app (D3 + Tableau Extensions API + settings UI)
  manifest.trex    ← Tableau extension manifest
  README.md
  docs/
    plan.md        ← architecture blueprint
    context.md     ← live session state
    tasks.md       ← checklist
```

## Deploy & Test
- **Deploy:** `git add index.html && git commit -m "..." && git push`
- **Live URL:** https://bengtcj.github.io/b_tableau_extensions/index.html
- **Test browser:** open URL with `?chart=donut` (no Tableau needed — falls back to sample data after 3s)
- **Test Tableau:** load manifest.trex via Extension object on a dashboard, configure via ⚙ button
- **No linter, no build tool**

## Architecture
- **Single file app** — all JS/CSS/HTML in `index.html`
- **Config** stored in Tableau Extensions Settings API (persists in .twb workbook file)
- **Chart router** — `renderChart()` dispatches to `renderDonut/Bar/Line/BAN()`
- **Design system** — `buildDesign()` returns per-element font/colour specs, rebuilt on each render
- **Tableau API** loaded dynamically (not in `<head>`) — falls back to sample data if unavailable
- **ResizeObserver** triggers re-render on container resize

## Chart Types
| Value | Chart |
|-------|-------|
| `donut` | Donut with BAN + legend |
| `bar` | Sorted bar, selected brand highlighted |
| `line` | Multi-brand line, selected on top |
| `ban` | Single headline number |

## Design System Constants
```javascript
FONT_TITLE = Baskerville italic       // BAN numbers
FONT_BODY  = Tableau Regular          // value labels, legend
FONT_LIGHT = Tableau Light            // brand names (always uppercase), axis labels
COL_LIGHT_GREY = #e6e6e6
COL_DARK_GREY  = #666666
COL_GRID       = #444444
```
Default selectedColor: `#e994a2` | neutralColor: `#7d7d7d` | bgColor: `transparent`

## Tableau Context
- **Platform:** Tableau Cloud (primary) + Desktop (authoring)
- **Data source:** Snowflake via BP Index Scores Extract
- **Key fields:** `Brand Name Upper`, `Raw Value`, `indicator_name`, `Quarter`
- **Parameter:** `Select Brand`
- **Allowlisted URL:** `https://bengtcj.github.io/b_tableau_extensions/index.html`

## Code Style
- No external dependencies except D3 (CDN) and Tableau Extensions API (CDN)
- No TypeScript, no bundler, no framework
- Inline styles on chart elements (D3 pattern) — CSS only for settings dialog
- All chart sizing uses `el.offsetWidth / el.offsetHeight` (not vw/vh — breaks in iframes)
- Brand names always `text-transform: uppercase` + `FONT_LIGHT`

## Anti-overengineering
Ultrathink first. Explore existing code. Make only minimal necessary changes.
Reuse what exists. No new abstractions unless explicitly asked.
This is a single HTML file — keep it that way unless there is a very strong reason.

## Experimental loop (for Tableau-specific issues)
List options. Try option A as minimal proof of concept. Test it.
If it fails, note why, delete it, try B. Clean up failures.
Report only what worked and why. After each attempt tell me what to look for and what success looks like.

## End of session
At the end of every session, suggest updates to the MD files that reflect what changed:
- `context.md` — move completed items to "What's Working", update Known Issues
- `tasks.md` — tick completed items, add new ones discovered
- `plan.md` — update status table if chart types or architecture changed
- `client_brand_investigation.md` (or equivalent handoff doc) — mark resolved or add new findings

## Token discipline
- Use grep/head/tail for targeted checks, not full file reads
- Prove concepts in 10 lines before full implementation
- Summarise after acting, not during
- Do not re-read files already read this session
- Do not narrate every step while iterating
- Ask one clarifying question before writing, not five after
