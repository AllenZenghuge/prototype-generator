# Shot-Mode Prompt Builder

Processes UI screenshots into pixel-accurate HTML prototypes.

## Process

### Phase 0.5 — Page Isolation Rules (READ BEFORE ANY CODE CHANGE)

**These rules prevent cross-page breakage. Violating any of them is a hard stop.**

#### Shared File Modification Rules

| File | Allowed | Forbidden |
|------|---------|-----------|
| `framework.js` | Append new page's `buildHeader` branch ONLY | Modify existing branches, delete existing logic, change function signatures |
| `common.js` | Append new page-specific `navData` array ONLY | Delete/modify existing navData, change existing function signatures except `getPageName` (add new path) and `renderSidebar` navData selection |
| `common.css` | **NEVER modify** | All changes forbidden — page-specific styles go in the page's inline `<style>` |

#### Menu Isolation Protocol

When adding a new page's sidebar menu to `common.js`:

```
1. Read the sidebar structure from vision_result_sidebar_{ts}.json (from vision-analyzer Phase 0.5)
2. Create a new navData array: var {pageName}Nav = [ ... ];
3. Add the new page path to getPageName()
4. Update renderSidebar() navData selection to include the new page
5. DO NOT touch any other navData arrays
6. Run: node -c common.js  (MUST pass)
7. Start HTTP server, verify all existing pages still show correct menus
```

#### Before Modifying Any Shared File

```
1. Report to user: "Modifying {file}: {what}, affects {which pages}"
2. Wait for user confirmation
3. Make the change
4. Run node -c {file}
5. Test all existing pages
```

---

### Phase 0.6 — Baseline Check

Before generating a new page that might overwrite an existing one:

```
1. List prototype/output/pages/ — check if target filename already exists
2. If exists:
   a. Copy to prototype/output/pages/{name}-baseline-{YYYYMMDD}.html
   b. Report: "已有版本已保存为 {name}-baseline-{YYYYMMDD}.html"
3. NEVER overwrite files containing "baseline" in their name
4. Generate new version to the original filename
```

Baseline naming: `{page_name}-baseline-{YYYYMMDD}.html` (e.g., `settlement-form-baseline-20260720.html`)

---

### Phase 1: Pixel Analysis

Follow `shot-mode/vision-analyzer.md` to:
1. Read `prototype/prototype.config.json` for vision model config
2. For each screenshot in `prototype/shots/`:
   - Classify screenshot relationships (Phase 0.3 — 5 types)
   - [Form pages] Extract sidebar menu (Phase 0.5)
   - Base64 encode the image
   - Send to vision model with context-aware prompt
   - Parse the JSON response
   - Save result immediately (Phase 2 — mandatory)
3. Merge all results into `merged_analysis_{timestamp}.json`

### Phase 2: Semantic Mapping (screenshot-to-code)

Invoke `Skill` tool with `skill="screenshot-to-code"` and pass:
- The structured JSON from Phase 1
- The original screenshots

The screenshot-to-code skill will:
- Map raw components to semantic equivalents:
  - "220×32px blue rectangle #1677ff" → "32px primary button (Ant Design style)"
  - "Dark sidebar 220px wide" → "sidebar navigation with groups"
  - "Data grid with column headers" → "table with sortable columns"
- Identify the design pattern (dashboard, list+detail, form wizard, etc.)
- Suggest the appropriate HTML/CSS component structure

### Phase 3: Design System (Optional but Recommended)

If `prototype/规范.md` exists:
- Read `docs-mode/prompt-builder.md` and follow Phase 2 (Generate Design System)
- Call ui-ux-pro-max → frontend-design

If no spec doc:
- Derive design tokens from the pixel analysis:
  - Colors → CSS variables
  - Spacing → spacing scale
  - Font sizes → typography scale
- Apply frontend-design review to the derived tokens

### Phase 4: Conflict Resolution

Compare pixel analysis results with the design system (from spec or derived):

```
"If the screenshot shows {value} but the spec defines {spec_value} for {property}:
 - Mark as CONFLICT
 - Default behavior: screenshot value wins (we're recreating the screenshot)
 - If the spec value is a brand requirement (logo, brand color), flag for user:

 'Conflict detected:
  - Screenshot shows: primary color = {screenshot_hex}
  - Spec defines: primary color = {spec_hex}
  Which should be used in the prototype?
  1. Screenshot value (match the screenshot exactly)
  2. Spec value (use brand standard)
  Enter 1 or 2:'"
```

### Phase 5: Fill the Template

Read `docs-mode/template.md`. Fill all four layers combining:
1. Screenshot analysis (page structure, components, data)
2. Design system (tokens, component specs)
3. Spec doc values (if available, for constraints)

For the Content-Driven layer:
- Page list = one page per screenshot (or group related screenshots)
- If multiple screenshots form a flow (list → detail → form), note the navigation
- CONTEXT should reference "screenshot analysis from {N} screenshots"

### Phase 6: Present and Execute

Present the complete prompt as in docs-mode Phase 4. When user confirms "go", execute.

---

### Phase 7 — Mandatory Self-Check (DO NOT SKIP)

**Execute ALL 5 checks before presenting to user. Any FAIL must be fixed before delivery.**

#### Check 1: JS Syntax
```bash
node -c prototype/output/js/common.js && echo "PASS" || echo "FAIL"
node -c prototype/output/js/framework.js && echo "PASS" || echo "FAIL"
```

#### Check 2: Multi-Page Regression
- List all .html files in `prototype/output/pages/`
- For each existing page (not the newly generated one):
  - Confirm its `<script src=...>` references are intact
  - Confirm its corresponding menu item exists in common.js
  - Visit the page in browser and verify sidebar menu renders correctly

#### Check 3: §14 Template Skeleton
Read `prototype/规范.md` §14 and verify against the new page:
```
[ ] Sidebar: full-height, dark #001529, logo+search+menu+collapse button
[ ] Primary header: 40px tabs + grid icon + home icon + active tab + right-side icons (bell, gear, avatar)
[ ] Secondary header (if applicable): 40px status filter tabs
[ ] Table (if applicable): checkbox + row number + action icon buttons, number cols right-aligned
[ ] Table (if applicable): min-width for horizontal scroll
[ ] Bottom bar (if applicable): pagination, outside table wrapper
[ ] Page-specific styles use var(--hx-*) tokens
[ ] No duplicate UI elements from incremental edits
```

Note: Not all items apply to every page type. Mark N/A for non-applicable items.

#### Check 4: CSS Token Usage
```bash
TOKEN_COUNT=$(grep -c 'var(--hx-' prototype/output/pages/{new_page}.html)
HARDCODED_COLORS=$(grep -oP '#[0-9a-fA-F]{3,8}' prototype/output/pages/{new_page}.html | wc -l)
echo "Token refs: $TOKEN_COUNT, Hardcoded colors: $HARDCODED_COLORS"
# PASS if TOKEN_COUNT >= 20 AND HARDCODED_COLORS <= 3
```

#### Check 5: Control Height Compliance
```bash
grep -oP 'height:\s*\d+px' prototype/output/pages/{new_page}.html | sort -u
# FAIL if any height is NOT in [16, 24, 32, 36, 40, 48, 56, 64]
```

#### Self-Check Report Format
```
## Phase 7 自检报告
| # | 检查项 | 结果 | 详情 |
|---|--------|------|------|
| 1 | JS 语法 | PASS/FAIL | common.js + framework.js |
| 2 | 多页面回归 | PASS/FAIL | N existing pages verified |
| 3 | §14 模板 | PASS/FAIL | M/N items passed |
| 4 | Token 使用率 | PASS/FAIL | N refs, M hardcoded |
| 5 | 控件高度 | PASS/FAIL | All heights compliant / Found: [list] |

结论：[全部通过 / 需修复 N 项]
```

**If any check FAILs, fix the issue and re-run ALL checks. Only present to user after all 5 pass.**

---

### After Generation

After self-check passes, proceed to verifier.md for the full 13-check validation (including Check 0 JS syntax + Check 1-12).
