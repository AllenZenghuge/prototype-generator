---
title: prototype-generator Skill 实现计划
date: 2026-06-30
tags:
  - implementation-plan
  - skill
  - prototype
aliases:
  - prototype-generator 实现
  - 原型生成器开发计划
description: >
  基于已确认的设计 Spec，将 prototype-generator skill 拆分为 7 个 Task 逐文件实现。
  这是一个纯 Markdown 编写的 Claude Code Skill，每个 Task 对应一个 skill 子文件。
source: spec/20260630-prototype-generator-skill-design.md
---

# prototype-generator Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现一个 Claude Code Skill，支持 `/prototype:docs`（需求文档→HTML原型）和 `/prototype:shot`（截图→HTML原型）两种模式。

**Architecture:** Skill 由 7 个 Markdown 文件组成。入口 SKILL.md 路由到两个模式的 prompt-builder。每个 builder 按四层模板（不可变/值变量化/结构保持/内容驱动）构造 /goal 提示词。产物通过 Playwright + 视觉模型自动验证。

**Tech Stack:** Claude Code Skill (Markdown) + 外部 Skill 联动 (ui-ux-pro-max, frontend-design, screenshot-to-code) + Playwright MCP + Qwen3.7-Max

## Global Constraints

- 产出兼容 Axure/墨刀 HTML 导入（无复杂 JS 框架依赖）
- `/prototype:docs` 和 `/prototype:shot` 均须先执行 Step 0 依赖检查
- CONSTRAINTS 默认值使用 deepal `/goal` 模板原值，用户规范可覆盖
- 视觉模型默认 Qwen3.7-Max，可通过 `/prototype:config` 切换
- 所有文件输出到 `prototype/output/` 目录
- 验证阶段：docs-mode 12 项检查，shot-mode 额外 4 项像素检查

---

## File Structure

```
.claude/skills/prototype-generator/
├── SKILL.md                         # Task 1: 入口 + 路由 + Step 0 依赖检查
├── config.md                        # Task 2: 配置管理 + 首次使用向导
├── docs-mode/
│   ├── template.md                  # Task 3: 泛化 /goal 提示词模板（四层）
│   └── prompt-builder.md            # Task 4: docs-mode 完整流程
├── shot-mode/
│   ├── vision-analyzer.md           # Task 5: Qwen3-VL 像素分析指令
│   └── prompt-builder.md            # Task 6: shot-mode 完整流程
└── verifier.md                      # Task 7: 自动化验证指令
```

**责任划分：**

| 文件 | 职责 |
|------|------|
| SKILL.md | 命令解析、模式路由、Step 0 依赖检查、错误提示 |
| config.md | 读/写 prototype.config.json、首次向导、provider 切换 |
| docs-mode/template.md | 四层模板定义、变量占位符、CONSTRAINTS 默认值 |
| docs-mode/prompt-builder.md | 文档解析、调用 ui-ux-pro-max + frontend-design、填充模板 |
| shot-mode/vision-analyzer.md | Qwen3-VL API 调用、结构化输出格式定义 |
| shot-mode/prompt-builder.md | 截图→分析→screenshot-to-code→设计系统→填充模板 |
| verifier.md | Playwright 截图、检查清单、像素对比、验证报告 |

---

### Task 1: SKILL.md — 入口 + 路由 + 依赖检查

**Files:**
- Create: `.claude/skills/prototype-generator/SKILL.md`

**Interfaces:**
- Produces: 三个命令入口 `/prototype:docs`, `/prototype:shot`, `/prototype:config`
- Produces: Step 0 依赖检查逻辑，调用 config.md 读配置，路由到对应 builder

- [ ] **Step 1: 创建 SKILL.md 骨架**

写入以下内容到 `.claude/skills/prototype-generator/SKILL.md`：

```markdown
---
name: prototype-generator
description: Generate interactive HTML prototypes from requirements docs + design specs (/prototype:docs) or from screenshots (/prototype:shot). Output compatible with Axure/墨刀 import.
---

# Prototype Generator

根据需求文档或 UI 截图，生成可交互的 HTML 原型页面，兼容 Axure/墨刀导入。

## Commands

| Command | Purpose |
|---------|---------|
| `/prototype:docs` | Requirements doc + design spec → HTML prototypes |
| `/prototype:shot` | UI screenshots → pixel-accurate HTML prototypes |
| `/prototype:config` | View/edit configuration (vision model, specs, verification) |

## Step 0 — Dependency Check (runs before either mode)

Before executing `/prototype:docs` or `/prototype:shot`, verify these skills are installed:

```
Required skills:
  - ui-ux-pro-max      → design system generation
  - frontend-design    → design review and uniqueness check
  - screenshot-to-code → [shot-mode only] screenshot semantic analysis

Check method:
  Use the Skill tool to attempt invoking each skill. If invocation fails with "not found", the skill is missing.

If any skill is missing, stop and output:
  "Missing dependency skills: {skill_names}.
   Install them:
   - ui-ux-pro-max: ask Claude 'install ui-ux-pro-max skill'
   - frontend-design: ask Claude 'install frontend-design skill'
   - screenshot-to-code: ask Claude 'install screenshot-to-code skill'
   Then re-run your command."
```

## /prototype:docs — Document-Driven Mode

**Input required:**
- `prototype/规范.md` — UI design spec (required)
- `prototype/docs/*.md` — Requirements documents (required)

**Execution flow:**

1. Read `config.md` to learn configuration management and directory structure
2. Verify `prototype/` directory exists; if not, create it with subdirectories (`docs/`, `shots/`, `output/`)
3. Verify `prototype/规范.md` exists; if not, prompt user to add it
4. Verify `prototype/docs/` has `.md` files; if not, prompt user to add them
5. Read `docs-mode/prompt-builder.md` and follow its instructions exactly
6. Read `docs-mode/template.md` for the /goal prompt template structure
7. After generation completes, read `verifier.md` and follow its instructions

## /prototype:shot — Screenshot-Driven Mode

**Input required:**
- `prototype/规范.md` — UI design spec (optional but recommended)
- `prototype/shots/*.png` — Screenshots (required)

**Execution flow:**

1. Read `config.md` to learn configuration and check vision model setup
2. Verify `prototype/` directory; create if missing
3. Verify `prototype/shots/` has screenshots; if not, prompt user
4. Read `shot-mode/vision-analyzer.md` and follow its pixel analysis instructions
5. Read `shot-mode/prompt-builder.md` and follow its instructions exactly
6. Read `docs-mode/template.md` for the /goal prompt template structure
7. After generation completes, read `verifier.md` and follow its instructions

## /prototype:config — Configuration Management

Read `config.md` and follow its instructions for the requested config operation.

## Error Handling

See `config.md` for configuration errors. For execution errors:

| Scenario | Action |
|----------|--------|
| Missing input files | Prompt user with exact paths expected |
| Spec doc unparseable | Stop, list sections that couldn't be parsed |
| Requirements have unclear page list | Stop, list detected pages, ask user to confirm |
| Screenshot quality too low | Prompt user for higher resolution (2x recommended) |
| Vision model API failure (shot-mode) | Fall back to spec-only generation, skip pixel analysis |
| Screenshot conflicts with spec | List conflicts, ask user which takes priority |
| >50 pages detected | Split into batches of ≤15 pages |
| Playwright unavailable | Fall back to manual verification checklist |
```

- [ ] **Step 2: 验证文件写入成功**

```bash
wc -l .claude/skills/prototype-generator/SKILL.md
```

Expected: ~80 lines, file exists.

---

### Task 2: config.md — 配置管理

**Files:**
- Create: `.claude/skills/prototype-generator/config.md`

**Interfaces:**
- Consumes: 无
- Produces: `readConfig()` 逻辑 — 读取 `prototype/prototype.config.json`
- Produces: `writeConfig()` 逻辑 — 写入配置
- Produces: `firstTimeWizard()` 逻辑 — 首次使用引导

- [ ] **Step 1: 创建 config.md**

```markdown
# Config Management

## Configuration File

`prototype/prototype.config.json` stores all skill configuration. If the file does not exist, run the first-time wizard.

### Default config values:

```json
{
  "vision_model": {
    "provider": "qwen3.7-max",
    "api_key": "",
    "base_url": "https://ws-udt9j53khqthrs7z.cn-beijing.maas.aliyuncs.com/compatible-mode/v1",
    "model_name": "qwen3.7-max-2026-06-08"
  },
  "default_spec": "./规范.md",
  "output_dir": "./output",
  "resolution": ["1366x768", "1440x900", "1920x1080"],
  "verification": {
    "enabled": true,
    "engine": "playwright"
  }
}
```

## First-Time Wizard

When `prototype/prototype.config.json` does not exist:

1. Create the `prototype/` directory with subdirectories: `docs/`, `shots/`, `output/css/`, `output/js/`, `output/pages/`
2. Ask the user to configure the vision model:

```
"First time setup! Which vision model provider for screenshot analysis?
1. Qwen3.7-Max (default, needs API key)
2. Claude Vision (built-in, no API key needed)
3. GPT-4o (needs API key)

Enter number:"
```

3. If provider needs API key, prompt: `"Enter your API key for {provider}:"`
4. Write `prototype/prototype.config.json` with the selected values
5. Confirm: `"Configuration saved to prototype/prototype.config.json. Place your UI spec as prototype/规范.md to get started."`

## /prototype:config Operations

### View current config
When called with no arguments, read and display `prototype/prototype.config.json`:
```
"Current configuration:
  Vision model: {provider} ({model_name})
  Default spec: {path}
  Output dir: {path}
  Resolutions: {list}
  Verification: {'enabled' / 'disabled'} (engine: {engine})"
```

### Change vision model: `/prototype:config vision`
1. Show current vision model config
2. Ask: `"Change to which provider? (qwen3.7-max / claude-vision / gpt-4o)"`
3. If API key needed, prompt for it
4. Update `prototype/prototype.config.json`

### Change spec path: `/prototype:config spec`
1. Prompt for new spec path
2. Update `default_spec` in config

### Toggle verification: `/prototype:config verify on|off`
1. Set `verification.enabled` to `true` or `false`
2. Confirm: `"Verification {'enabled' / 'disabled'}."`

### Change verification engine: `/prototype:config engine`
1. Show current engine
2. Ask: `"Switch to which engine? (playwright / agentcloak)"`
3. Update `verification.engine` in config
```

- [ ] **Step 2: 验证文件**

```bash
grep -c "prototype.config.json" .claude/skills/prototype-generator/config.md
```

Expected: >= 3 matches.

---

### Task 3: docs-mode/template.md — 泛化 /goal 提示词模板

**Files:**
- Create: `.claude/skills/prototype-generator/docs-mode/template.md`

**Interfaces:**
- Produces: 四层模板定义，每层标注占位符变量 `{PLACEHOLDER_NAME}`
- Produces: CONSTRAINTS 13 条的完整文本（默认值版）

- [ ] **Step 1: 创建 template.md**

```markdown
# /goal Prompt Template (Generalized)

This template is the core of the prototype generator. It is derived from the verified deepal `/goal` template and organized into four layers.

## Layer System

| Layer | Sections | Strategy |
|-------|----------|----------|
| Layer 1 — Immutable | STOP RULES, OUTPUT, VERIFY | Use verbatim, never modify |
| Layer 2 — Parameterized | CONSTRAINTS | Fixed rules, variable values from spec |
| Layer 3 — Structure-Preserving | PLAN, DONE WHEN, PRIORITY | Fixed skeleton, page counts/names vary |
| Layer 4 — Content-Driven | CONTEXT, Page List, Navigation | Everything extracted from input docs |

---

## Layer 1 — Immutable (USE VERBATIM)

### STOP RULES
```
如果 PRD 功能清单不明确，停止并询问。
如果参考文件不存在或风格无法识别，停止并询问。
不得添加 PRD 中未提及的页面或功能。
遇到 CSS Token 冲突时，优先遵循 UI 规范文档。
```

### OUTPUT
```
提供文件清单（所有 HTML 和 CSS 文件路径）。
提供页面跳转关系说明。
说明复用的公共组件和样式。
列出未实现的交互功能（如真实数据加载、表单提交）。
```

### VERIFY
```
检查每个页面的布局是否符合 PRD 功能清单。
检查配色、字体、间距是否与参考文件一致。
逐一点击所有导航链接，验证跳转是否正确。
检查表格数值列是否右对齐。
检查表单输入框聚焦、按钮 hover 效果是否正常。
检查 CSS 变量是否被正确引用。
在 {RESOLUTION_1}、{RESOLUTION_2}、{RESOLUTION_3} 分辨率下测试布局。
```
Note: {RESOLUTION_1/2/3} are the only variables in VERIFY — extracted from prototype.config.json.

---

## Layer 2 — Parameterized (FIXED RULES, VARIABLE VALUES)

### CONSTRAINTS

Default values (used when no spec doc provided):
```
不得实现后端 API 调用或真实业务逻辑。
使用模拟数据，尽量真实。
必须遵循 {SPEC_PATH} 的 CSS Token 和组件约束。
CSS 必须分离为独立文件（common.css 公共样式 + 各页面独立样式）。
每个页面对应一个独立的 HTML 文件。
必须使用 CSS 变量定义颜色、尺寸、间距，不得散落硬编码。
控件高度符合 {CONTROL_HEIGHTS}，间距落在 {SPACING_STEP} 步进体系。
表格数值列、金额列、百分比列右对齐。
日期格式 {DATE_FORMAT}，日期时间格式 {DATETIME_FORMAT}。
金额使用千分位，百分比保留 2 位小数。
表单需要基础交互：输入框聚焦效果、按钮 hover 效果、表单验证提示样式。
菜单/导航点击必须能够跳转到对应页面。
状态使用 Tag 展示（草稿、审核中、已入库、已归档等）。
```

Parameter defaults:
```
{SPEC_PATH} = "./规范.md"
{CONTROL_HEIGHTS} = "24/32/40px"
{SPACING_STEP} = "4px"
{DATE_FORMAT} = "yyyy-mm-dd"
{DATETIME_FORMAT} = "yyyy-mm-dd HH:mm:ss"
```

When a spec doc is provided, extract these values from the spec and override defaults.

---

## Layer 3 — Structure-Preserving (FIXED SKELETON)

### PRIORITY
```
1. 完成公共布局框架（顶部导航、侧边栏、面包屑、页脚）
2. 完成 common.css 公共样式文件（CSS Token、通用组件样式）
3. 按一级导航优先级创建页面：
   {P0_PAGES}
   {P1_PAGES}
   {P2_PAGES}
   {P3_PAGES}
4. 实现导航菜单和页面跳转链接
5. 添加必要的 CSS 交互效果
```
Note: {P0_PAGES} through {P3_PAGES} are extracted from the requirements doc's navigation hierarchy.

### PLAN
```
第一步：分析 {REFERENCE_FILES} 的布局结构、配色方案、组件风格
第二步：创建项目目录结构（pages/、css/、images/）
第三步：创建 common.css，包含 CSS Token 和通用组件样式
第四步：创建公共布局模板（顶部导航、侧边栏、内容区）
第五步：按优先级创建各页面 HTML 文件（共 {TOTAL_PAGES} 个）
第六步：实现导航菜单和页面跳转
第七步：添加表格、表单、按钮等组件的交互效果
第八步：检查所有页面跳转链接和样式一致性
```
Note: {REFERENCE_FILES} defaults to "规范文档和设计系统"; {TOTAL_PAGES} from requirements.

### DONE WHEN
```
所有页面 HTML 文件创建完成（约 {PAGE_COUNT} 个）。
所有页面可通过左侧导航菜单跳转。
视觉风格与 {SPEC_NAME} 一致。
表格、表单、按钮有真实的交互效果。
CSS 文件独立分离，使用 CSS 变量。
所有链接跳转正确无误。
符合 {SPEC_NAME} 的检查清单。
```

---

## Layer 4 — Content-Driven (FROM INPUT DOCUMENTS)

### CONTEXT
```
项目路径：{PROJECT_PATH}
参考样式文件：{REFERENCE_STYLES}
UI 规范文档：{SPEC_PATH}
页面类型：{PAGE_TYPE}
设计风格：{DESIGN_STYLE}
推荐组件语义：{COMPONENT_SEMANTICS}
目标分辨率：{TARGET_RESOLUTIONS}
```

### Page List
```
{Navigation hierarchy extracted from requirements doc}
```

### Navigation
```
{Page jump relationships extracted from requirements doc}
```
```

- [ ] **Step 2: 验证模板包含所有四层**

```bash
grep -c "Layer" .claude/skills/prototype-generator/docs-mode/template.md
```

Expected: >= 4 (four layers defined).

---

### Task 4: docs-mode/prompt-builder.md — 文档驱动构造器

**Files:**
- Create: `.claude/skills/prototype-generator/docs-mode/prompt-builder.md`

**Interfaces:**
- Consumes: `prototype/规范.md`, `prototype/docs/*.md`, `template.md`
- Consumes: ui-ux-pro-max skill (via Skill tool), frontend-design skill (via Skill tool)

- [ ] **Step 1: 创建 prompt-builder.md**

```markdown
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
/ goal

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
```

- [ ] **Step 2: 验证文件**

```bash
grep -c "### Phase" .claude/skills/prototype-generator/docs-mode/prompt-builder.md
```

Expected: 4.

---

### Task 5: shot-mode/vision-analyzer.md — 像素分析指令

**Files:**
- Create: `.claude/skills/prototype-generator/shot-mode/vision-analyzer.md`

**Interfaces:**
- Produces: Qwen3-VL API 调用指令 + 结构化输出 JSON Schema

- [ ] **Step 1: 创建 vision-analyzer.md**

```markdown
# Vision Analyzer — Pixel-Level Screenshot Analysis

Sends screenshots to a configurable vision model for structured UI extraction.

## Model Selection

Read `prototype/prototype.config.json` → `vision_model`. Determine provider:

| Provider | API Call Method |
|----------|----------------|
| qwen3.7-max | OpenAI-compatible API via `Bash` tool with curl |
| claude-vision | Read the image file and pass to Claude's vision capability directly |
| gpt-4o | OpenAI API via `Bash` tool with curl |

## Qwen3.7-Max API Call (Default)

When provider is `qwen3.7-max`:

```bash
# For each screenshot in prototype/shots/:
curl -s "https://ws-udt9j53khqthrs7z.cn-beijing.maas.aliyuncs.com/compatible-mode/v1/chat/completions" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.7-max-2026-06-08",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "image_url", "image_url": {"url": "data:image/png;base64,{BASE64_IMAGE}"}},
          {"type": "text", "text": "{ANALYSIS_PROMPT}"}
        ]
      }
    ],
    "max_tokens": 4096
  }'
```

### ANALYSIS_PROMPT (sent to vision model)

```
Analyze this UI screenshot and output a structured JSON description. Be precise with pixel values.

Output JSON schema:
{
  "layout": {
    "type": "left-right | top-bottom | single-column | left-center-right",
    "header": { "height": "px", "background_color": "hex", "elements": ["..."] },
    "sidebar": { "width": "px", "background_color": "hex", "position": "left|right|none" },
    "content": { "background_color": "hex" },
    "footer": { "height": "px", "visible": true|false }
  },
  "components": [
    {
      "type": "table | form | button | input | select | tag | tab | card | modal | drawer | stat-card | breadcrumb | pagination | chart | tree | timeline",
      "label": "human-readable name",
      "position": { "x": "px", "y": "px", "width": "px", "height": "px" },
      "style": {
        "background_color": "hex",
        "text_color": "hex",
        "border_color": "hex",
        "font_size": "px",
        "font_weight": "normal|medium|bold",
        "border_radius": "px",
        "padding": "px",
        "margin": "px"
      },
      "children": ["nested component descriptions"],
      "state": "default|hover|focus|active|disabled|error"
    }
  ],
  "colors": {
    "primary": "hex",
    "background": "hex",
    "surface": "hex",
    "text_primary": "hex",
    "text_secondary": "hex",
    "border": "hex",
    "success": "hex",
    "warning": "hex",
    "error": "hex"
  },
  "typography": {
    "heading_size": "px",
    "body_size": "px",
    "caption_size": "px",
    "font_family": "system|serif|mono"
  },
  "spacing": {
    "component_gap": "px",
    "section_padding": "px",
    "grid_columns": number
  },
  "data_examples": {
    "table_rows": ["example row 1 content", "example row 2 content"],
    "labels": ["visible text labels found in screenshot"],
    "numbers": ["number values with their format (currency/percentage/count)"]
  }
}

Important:
- ALL color values must be in hex format (#RRGGBB or #RRGGBBAA)
- ALL measurements in px
- If a component's state is not default, note it
- Include visible text content as data_examples for realistic mock data
- Note any icons or images with descriptions
```

### Parsing the Response

Parse the JSON response. For each component:
1. Map raw pixel values to the nearest design system token
2. Flag values that don't align to 4px grid
3. Note components that match spec-defined patterns (e.g., "this 32px blue button matches primary button spec")

### Handling Multiple Screenshots

If `prototype/shots/` contains multiple screenshots:
- Process each independently
- Cross-reference: identify shared components (same sidebar, same header)
- Deduplicate: shared components only need to be generated once
- Generate a unified page list from all screenshots

## Error Handling

| Scenario | Action |
|----------|--------|
| API key missing | Prompt: "Vision model API key not configured. Run /prototype:config vision to set it." |
| API returns error | Show error message. Offer fallback: "Vision analysis failed. Continue with spec-only generation? (y/n)" |
| JSON parse failure | Show raw response excerpt. Ask: "Could not parse vision model output. Retry? (y/n)" |
| Image too large | Compress to < 5MB before base64 encoding |
| Zero components detected | Ask: "No UI components detected in screenshot. Is this a valid UI screenshot? (y/n)" |
```

- [ ] **Step 2: 验证包含 API 调用代码和 JSON Schema**

```bash
grep -c "curl" .claude/skills/prototype-generator/shot-mode/vision-analyzer.md
grep -c "json" .claude/skills/prototype-generator/shot-mode/vision-analyzer.md
```

Expected: >= 1 curl, >= 2 json.

---

### Task 6: shot-mode/prompt-builder.md — 截图驱动构造器

**Files:**
- Create: `.claude/skills/prototype-generator/shot-mode/prompt-builder.md`

**Interfaces:**
- Consumes: vision-analyzer.md, screenshot-to-code skill, docs-mode/template.md
- Consumes: ui-ux-pro-max, frontend-design skills

- [ ] **Step 1: 创建 prompt-builder.md**

```markdown
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
```

- [ ] **Step 2: 验证引用链**

```bash
grep -c "vision-analyzer.md" .claude/skills/prototype-generator/shot-mode/prompt-builder.md
grep -c "screenshot-to-code" .claude/skills/prototype-generator/shot-mode/prompt-builder.md
```

Expected: >= 1 each.

---

### Task 7: verifier.md — 自动化验证

**Files:**
- Create: `.claude/skills/prototype-generator/verifier.md`

**Interfaces:**
- Consumes: `prototype/output/` generated files
- Consumes: Playwright MCP or agentcloak (from config)
- Consumes: vision model (for shot-mode pixel comparison)

- [ ] **Step 1: 创建 verifier.md**

```markdown
# Verification

Automated QA for generated HTML prototypes.

## Prerequisites

Read `prototype/prototype.config.json`:
- If `verification.enabled` is false, skip verification and only output file list.
- `verification.engine`: "playwright" or "agentcloak"

## docs-mode Verification (12 Checks)

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
- Passed: {N}/12
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
```

- [ ] **Step 2: 验证检查项数量**

```bash
grep -c "\[ \]" .claude/skills/prototype-generator/verifier.md
```

Expected: >= 16 (12 docs-mode + 4 shot-mode checkboxes).

---

## Final Validation

- [ ] **Validate: All files exist and are non-empty**

```bash
for f in \
  .claude/skills/prototype-generator/SKILL.md \
  .claude/skills/prototype-generator/config.md \
  .claude/skills/prototype-generator/docs-mode/template.md \
  .claude/skills/prototype-generator/docs-mode/prompt-builder.md \
  .claude/skills/prototype-generator/shot-mode/vision-analyzer.md \
  .claude/skills/prototype-generator/shot-mode/prompt-builder.md \
  .claude/skills/prototype-generator/verifier.md; do
  echo "$f: $(wc -l < "$f") lines"
done
```

Expected: All >= 30 lines, no errors.

- [ ] **Validate: Cross-references between files**

```bash
# SKILL.md should reference all sub-files
grep -c "config.md\|docs-mode/\|shot-mode/\|verifier.md" .claude/skills/prototype-generator/SKILL.md

# Expected: >= 5
```
```
