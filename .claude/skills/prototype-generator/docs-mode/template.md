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
