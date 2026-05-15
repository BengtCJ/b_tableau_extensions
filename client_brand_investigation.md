# Client Brand Selection — Investigation Handoff

## Goal
On extension load, the **client brand** (the one named in the `Select Client Brand` Tableau parameter) should render in pink (`CONFIG.selectedColor = '#e994a2'`); all other brands grey (`CONFIG.neutralColor = '#7d7d7d'`). Clicks should override and pin the clicked brand pink — that part works.

The failure case: on load the extension highlights the wrong brand (typically `data[0]`, the first brand alphabetically), not the parameter's current value.

## What's been tried

### 1. Read parameter directly, exact-string match
```js
const param = allParams.find(p => p.name === CONFIG.parameter);
if (param) selectedBrand = param.currentValue.formattedValue || param.currentValue.value;
```
**Result:** parameter found, but `selectedBrand === d.name` never true → falls through, wrong brand pink.

### 2. Case-insensitive lookup against `data[]`
```js
const raw = param.currentValue.formattedValue || param.currentValue.value;
const match = data.find(d => d.name.toLowerCase() === raw.toLowerCase());
selectedBrand = match ? match.name : raw;
```
**Result:** still wrong. Either the param's `currentValue` is not the brand string we expect, or the strings differ in more than case (whitespace, accents, punctuation).

### 3. Confirmed parameter is found (not a name mismatch)
Console showed `Loaded CONFIG: {... "parameter":"Select Client Brand" ...}` and the parameter exists in the dashboard. The "param not found" warn branch never fires.

### 4. Verified extension worksheet & data
- Worksheet: `vis_sov`
- `brandField`: `Brand Name Upper` (this returns ALL CAPS values like `ILLY COFFEE`)
- `valueField`: `SUM(Raw Value)`
- Parameter values list: `['Illy Coffee', 'BRAND 2']` (mixed case, not all caps)

So `data[].name === "ILLY COFFEE"` while `param.currentValue.value === "Illy Coffee"`. Case-insensitive match should bridge this — but it isn't. Need to log actual values to confirm.

## NEW INFO — likely smoking gun

The user has a Tableau calculated field used for color encoding:
```
IF [Brand] = [Select Client Brand]
THEN "Selected"
ELSE "Other"
END
```
This compares `[Brand]` (the original mixed-case dimension) against `[Select Client Brand]` (the parameter). For this calc to ever return "Selected" in Tableau itself, the parameter value must match `[Brand]` exactly — i.e., the param holds `"Illy Coffee"`, not `"ILLY COFFEE"`.

The extension is reading `[Brand Name Upper]` not `[Brand]`. So the names compared in the extension (`ILLY COFFEE`) and the names stored in the parameter (`Illy Coffee`) are case-different.

The case-insensitive fallback (#2 above) **should** have worked. The fact that it didn't means one of:
- `param.currentValue.formattedValue` is something unexpected (maybe the formatted display value is different from the raw value, or null)
- The brand from `[Brand Name Upper]` has trailing whitespace, an em-dash vs hyphen, accented characters, or other non-case difference
- `data` is empty when the lookup runs (timing issue with `getSummaryDataAsync` returning before parameter resolves)

## Files
- `index.html` — single-file extension. Relevant function: `fetchAndRender` (around line ~510). Uses `_selectedBrand` module-level state.
- `context.md` — project state
- The repo: https://github.com/BengtCJ/b_tableau_extensions, hosted at https://bengtcj.github.io/b_tableau_extensions/index.html

## Suggested first steps for the new session

1. **Add diagnostic logging** to `fetchAndRender`:
   ```js
   console.log('PARAM raw:', JSON.stringify(param?.currentValue));
   console.log('DATA names:', data.map(d => JSON.stringify(d.name)));
   ```
   Get the user to paste the console output. This will reveal the *exact* string contents and any hidden characters.

2. **Check the parameter's data type** — `param.dataType` and `param.allowableValues`. If it's a list parameter, the `currentValue` shape might differ.

3. **Consider switching the extension to read `[Brand]` instead of `[Brand Name Upper]`** so the values naturally match the parameter. Uppercase rendering can be done in CSS via `text-transform: uppercase` (already applied in some chart axes). This avoids the case-mismatch entirely.

4. **Or add a second column to the data fetch**: include `[Brand]` alongside `[Brand Name Upper]` and match against the original-case column. Display the upper version, match using the original.

5. **Sanity check timing:** confirm `getSummaryDataAsync()` resolves before `getParametersAsync()`, and that `data` is non-empty when the lookup runs.

## What NOT to retry without new evidence
- Don't re-add `ignoreSelection: true` to `getSummaryDataAsync` — it caused only one brand to return.
- Don't loop `selectMarksByValueAsync` over `CONFIG.worksheet` — it persists selection on the data sheet and the next load returns one brand only.
- Don't remove the `ParameterChanged` listener — that's the only legit re-render trigger left.
