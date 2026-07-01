# Docs-Mode Prompt Builder

Constructs the complete /goal prompt by parsing input documents and filling the four-layer template.

## Process

### Phase 1: Parse Input Documents

#### 1.1 Parse the Spec Doc (`prototype/规范.md`)

Read the spec document and extract:

```
CSS Tokens:
  -- Colors: primary, success, warning, error, info, bg-base, bg-layout, text, text-secondary, border, split
  -- Font: font-family, font-size, font-size-sm/lg/xl, line-height, font-weight
  -- Sizes: control-height (xs/sm/default/lg), spacing tokens (xxs/xs/sm/ms/md/lg/xl/xxl), radius
  -- Shadows: shadow, shadow-tertiary, focus-primary/error/warning

Component Constraints:
  -- Button: sizes, variants, states
  -- Table: header style, row height, alignment rules
  -- Form: label alignment, input sizes, validation states
  -- Modal/Drawer: dimensions, padding, button order
  -- Tag: color variants, size
  -- Tabs: style, active indicator
  -- Pagination: layout, size

Checklist:
  -- All items from spec's "HTML生成检查清单" section

Brand Elements:
  -- Logo path/description
  -- Brand colors (beyond the token system)
  -- Organization name (for header)

Parametrized Values (for CONSTRAINTS):
  -- control_heights: extract from spec, default "24/32/40px"
  -- spacing_step: extract from spec, default "4px"
  -- date_format: extract from spec, default "yyyy-mm-dd"
  -- datetime_format: extract from spec, default "yyyy-mm-dd HH:mm:ss"
```

If the spec doc has a "逐页来源索引" or similar table, treat it as metadata; the spec content is what matters.

#### 1.2 Parse Requirements Docs (`prototype/docs/*.md`)

Read all `.md` files in `prototype/docs/`. For each document, extract:

```
Navigation Hierarchy:
  Level 1: navigation groups (e.g., "工作台", "资产数据库", "资产交易")
  Level 2: pages under each group, with their type (list/detail/form/dashboard/monitor)

Page Types:
  - List page: filter area + toolbar + table + pagination
  - Detail page: summary header + tabs + metrics + timeline
  - Form page: steps or anchor nav + dynamic tables + validation
  - Dashboard: stat cards + charts + quick entries
  - Monitor page: status indicators + log table

Jump Relationships:
  - Sidebar click → open/close group → navigate to page
  - List row "查看" → detail page
  - List row "编辑/新增" → form page
  - Detail breadcrumb → back to list
  - Dashboard quick entry → target page

Field Lists (for tables and forms):
  - Table columns: field name, alignment (left/right/center), type (text/number/date/tag/action)
  - Form fields: label, type (input/select/date/upload), required, validation

Status Flows:
  - Status names and their Tag color mapping
  - Transition rules (which buttons appear in which status)
```

If the requirements doc's page list is unclear, stop and present detected pages for user confirmation:

```
"Detected {N} navigation groups and {M} pages. Does this look correct?

{Navigation tree preview}

Reply 'yes' to continue, or specify corrections."
```

### Phase 2: Generate Design System

#### 2.1 Call ui-ux-pro-max

Invoke `Skill` tool with `skill="ui-ux-pro-max"` and args describing the project:

```
"Product type: {extracted from requirements — e.g. 'enterprise asset management dashboard'}
 Style keywords: {extracted from spec — e.g. 'information-dense, stable, restrained, scannable'}
 Generate a complete design system with colors, typography, component recommendations, and UX checklist.
 Focus on the following page types: {list of page types extracted from requirements}."
```

Extract from ui-ux-pro-max response:
- Recommended color palette (map to CSS variable names)
- Font pairings (heading + body)
- Component style recommendations (table density, form layout, button hierarchy)
- UX checklist relevant to detected page types
- Any anti-pattern warnings

#### 2.2 Call frontend-design Review

Invoke `Skill` tool with `skill="frontend-design"` and args:

```
"Review this design system for uniqueness and personality:
 {paste ui-ux-pro-max output summary}

 Check against the three AI design traps:
 1. Warm cream background (#F4F1EA with terracotta)
 2. Near-black with single neon accent
 3. Newspaper-style (thin lines, zero radius, dense columns)

 For this {product_type} system:
 - What is the signature element (the one thing users remember)?
 - Does the typography have personality, or is it a neutral container?
 - Apply Chanel principle: what can we remove without losing expression?

 Provide 2-3 specific, actionable adjustments."
```

Incorporate frontend-design feedback into the design system before filling the template.

### Phase 3: Fill the Template

Read `docs-mode/template.md`. Fill each layer:

**Layer 1 (Immutable):** Copy verbatim from template. Only substitute resolution values from config.

**Layer 2 (Parameterized):** Copy all 13 constraint rules. Substitute `{SPEC_PATH}`, `{CONTROL_HEIGHTS}`, `{SPACING_STEP}`, `{DATE_FORMAT}`, `{DATETIME_FORMAT}` with values extracted from spec (or defaults).

**Layer 3 (Structure-Preserving):** Fill PRIORITY P0-P3 pages based on requirements navigation hierarchy. Fill PLAN with actual page count. Fill DONE WHEN with spec name and page count.

**Layer 4 (Content-Driven):** Populate CONTEXT from extracted values. Insert full page list and navigation relationships.

### Phase 4: Present and Execute

Output the complete /goal prompt in a markdown code block:

````
/goal

{complete filled prompt}
````

Then state:
```
"This is the prompt that will be executed. Review it and reply 'go' to execute, or describe any changes needed."
```

When user confirms:
1. Execute the /goal prompt as a single instruction
2. Claude will generate all files under `prototype/output/`
3. After generation completes, proceed to verifier.md
