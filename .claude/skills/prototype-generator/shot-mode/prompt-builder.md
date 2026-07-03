# Shot-Mode Prompt Builder

Processes UI screenshots into pixel-accurate HTML prototypes.

## Process

### Phase 1: Pixel Analysis

Follow `shot-mode/vision-analyzer.md` to:
1. Read `prototype/prototype.config.json` for vision model config
2. For each screenshot in `prototype/shots/`:
   - Base64 encode the image
   - Send to vision model with the ANALYSIS_PROMPT from vision-analyzer.md
   - Parse the JSON response
3. Collect all component descriptions into a unified structure

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

### After Generation

After files are generated, proceed to verifier.md with the additional `shot-mode` verification steps (pixel-level comparison).

### Post-Generation Self-Check (NEW — 2026-07-03)

Before handing off to user, run this mandatory self-check against the page template in the spec doc:

**Read `prototype/规范.md` §14 (标准页面模板) and verify:**
```
[ ] Sidebar: full-height, dark #001529, logo+search+menu+collapse button
[ ] Primary header: tabs + grid icon + home icon + active tab + right-side icons (bell, gear, avatar)
[ ] Secondary header: status filter tabs (草稿/流程中/所有单据)
[ ] Filter bar: sticky top, left/right split layout
[ ] Table: checkbox column + row number column + action icon buttons (6 icons, no text links)
[ ] Table: min-width for horizontal scroll, number columns right-aligned
[ ] Bottom bar: 合计金额 left + pagination right, same row, outside table wrapper
[ ] No duplicate UI elements (check for legacy code from incremental edits)
```

**If any check fails**, fix before presenting to user. This prevents the most common iteration patterns discovered in shot-mode testing.
