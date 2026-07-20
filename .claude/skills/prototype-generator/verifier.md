# Verification

Automated QA for generated HTML prototypes.

## Prerequisites

Read `prototype/prototype.config.json`:
- If `verification.enabled` is false, skip verification and only output file list.
- `verification.engine`: "playwright" or "agentcloak"

## docs-mode Verification (13 Checks)

### Check 0: JS Syntax (no browser needed — run first)

Before any other checks, validate all shared JS files:

```
[ ] 0. JS syntax valid
    - Run: node -c prototype/output/js/common.js
    - Run: node -c prototype/output/js/framework.js
    - FAIL if any syntax error
    - Report the exact error message and line number
```

### Check 1-5: CSS Analysis (no browser needed)

Read `prototype/output/css/common.css` and verify:

```
[ ] 1. CSS variables defined and used
    - grep for ":root {" block with --hx-* variables
    - grep for "var(--hx-" usage in rules (not just definition)
    - FAIL if >5 hardcoded colors/spacing values found

[ ] 2. Font stack is system font
    - grep font-family, must contain -apple-system, BlinkMacSystemFont
    - FAIL if font-family imports external fonts without spec approval

[ ] 3. tabular-nums set for numbers
    - grep "font-variant-numeric.*tabular-nums"
    - FAIL if not found

[ ] 4. Control heights use token sizes
    - grep for "height:.*px" in component rules
    - FAIL if heights found that aren't 16/24/32/40px (or spec-defined)

[ ] 5. Spacing uses 4px grid
    - grep for padding/margin/gap px values
    - FAIL if >20% of values are not multiples of 4
```

### Check 6-9: Browser Verification (requires Playwright/agentcloak)

For each `.html` file in `prototype/output/pages/`:

```
1. Open the page in browser
2. For each resolution in config (1366x768, 1440x900, 1920x1080):
   a. Resize viewport
   b. Wait 500ms for layout
   c. Take screenshot → save as output/verify/{page}_{resolution}.png

[ ] 6. Table number columns right-aligned
    - Use browser JS: check text-align of th.col-num, td.col-amount, td.col-percent
    - FAIL if any are not 'right'

[ ] 7. Date/amount format correct
    - Use browser JS: extract visible text, regex-check for yyyy-mm-dd and thousand-separated numbers
    - FAIL if malformed dates or unformatted numbers >10% of sample

[ ] 8. All nav links clickable
    - Use browser JS: querySelectorAll('a[href]'), click each, verify no 404
    - FAIL if any link returns error or doesn't navigate

[ ] 9. No overflow at any resolution
    - Compare scrollWidth to viewport width, scrollHeight to viewport height
    - FAIL if horizontal overflow detected (scrollWidth > viewport width + 5px)
```

### Check 10-12: Visual Verification (screenshot analysis)

```
[ ] 10. Empty/loading/error/disabled states
     - Check CSS for :empty, .loading, .error, :disabled rule sets
     - WARN (not FAIL) if some states missing — manual review needed

[ ] 11. Modal/drawer has radius + shadow + close
     - Check CSS for .modal/.drawer: border-radius, box-shadow, close button
     - FAIL if any missing

[ ] 12. Colors use token values
     - Scan CSS for hardcoded hex colors outside :root block
     - FAIL if hex colors found that don't match spec tokens
```

### Report Format

```
## Verification Report

### Summary
- Passed: {N}/13
- Failed: {M}/12
- Warnings: {W}

### Failures
| # | Check | Detail |
|---|-------|--------|
| 4 | Control heights | Found 36px, 44px at pages/xxx.html:123 |

### Warnings
| # | Check | Detail |
|---|-------|--------|
| 10 | Disabled state | No :disabled styles found for .btn |

### Screenshots
- output/verify/page1_1366x768.png
- output/verify/page1_1440x900.png
- ...
```

## shot-mode Extra Verification (S1-S4)

After docs-mode checks, run pixel-level comparison:

For each screenshot in `prototype/shots/` and its corresponding generated page:

```
S1. Color accuracy
    - Extract primary color from original screenshot
    - Extract primary color from generated page screenshot
    - Calculate ΔE (CIE76 or simple RGB distance)
    - FAIL if ΔE > 5

S2. Component hierarchy match
    - Compare the DOM structure to the vision model's component list
    - FAIL if any component from screenshot is missing in generated page

S3. Font size accuracy
    - Measure font sizes in generated page via browser JS
    - Compare to vision model's extracted font sizes
    - FAIL if any > 1px deviation

S4. Spacing accuracy
    - Measure element positions in generated page
    - Compare to vision model's extracted positions
    - FAIL if any spacing > 2px deviation
```

### Shot-Mode Report Addendum

```
### Pixel Accuracy
| # | Check | Threshold | Actual | Status |
|---|-------|-----------|--------|--------|
| S1 | Primary color ΔE | <5 | 2.3 | PASS |
| S2 | Component hierarchy | 100% match | 8/9 | FAIL (missing: pagination) |
| S3 | Font size deviation | <1px | max 0.5px | PASS |
| S4 | Spacing deviation | <2px | max 3px | FAIL (sidebar padding) |

### Regeneration Decision
- Failures exceeding threshold: {N}
- If N > 3 or includes layout-level failure → auto-regenerate
- If N ≤ 3 and all minor → display report, ask user: "Regenerate to fix {N} issues? (y/n)"
```

## Fallbacks

| Failure | Fallback |
|---------|----------|
| Playwright MCP not available | Output manual checklist for user to verify |
| agentcloak not available when configured | Fall back to Playwright, then to manual |
| Browser can't open local files | Serve with `python3 -m http.server` in output/ dir |
| Vision model fails (shot-mode S1-S4) | Skip pixel checks, report: "Pixel verification skipped (vision model unavailable)" |
