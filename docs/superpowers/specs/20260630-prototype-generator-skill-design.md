---
title: 原型生成器 Skill 设计方案
date: 2026-06-30
tags:
  - skill
  - prototype
  - html-css
  - design-system
  - screenshot-to-code
aliases:
  - prototype-generator skill 设计
  - 原型生成 skill
description: >
  基于 deepal 项目已验证的 HTML 原型生成工作流，设计一个通用型 Claude Code skill，
  支持两种模式：(1) 文档驱动：需求文档 + UI 规范 → 可交互 HTML 原型，
  (2) 截图驱动：UI 截图 → 像素级 HTML 原型，产出兼容 Axure/墨刀导入。
  集成 ui-ux-pro-max、frontend-design、screenshot-to-code 三个外部 skill
  提升生成质量，并通过 Playwright + 视觉模型实现自动化验证闭环。
source: deepal 项目工作流分析 + Allen 需求
---

# 原型生成器 Skill 设计方案

> | 时间 | 修订内容 |
> |------|----------|
> | 2026-06-30 11:30 | 初稿，完整设计 |

---

## 1. 背景与目标

### 1.1 来源

本设计基于 [deepal 项目](../../../个人资料/虹信/项目/01深蓝/deepal/) 已验证的 HTML 原型生成工作流：

1. 准备需求文档（markdown） + UI 规范文档（markdown）
2. 根据需求+规范，参考 `/goal` 提示词模板构造生成指令
3. 在 Claude Code 中运行 `/goal`，AI 自动生成 HTML/CSS/JS 原型

该工作流已成功产出 40+ 页面的资产管理系统中后台原型，质量经过 Playwright 截图验证。

### 1.2 目标

将上述工作流固化为一个通用型 Claude Code Skill，实现：

| 能力 | 描述 |
|------|------|
| **docs-mode** | 根据需求文档 + 公司前端设计规范 → 生成可交互 HTML 原型 |
| **shot-mode** | 根据 UI 截图 → 生成像素级可编辑 HTML 原型 |

产出兼容 Axure RP/墨刀等主流原型工具的 HTML 导入，可直接用于需求评审。

---

## 2. 架构总览

### 2.1 项目目录结构

```
{项目根目录}/prototype/
├── 规范.md                          # UI 设计规范文档（docs-mode 必选 / shot-mode 可选）
├── docs/                            # docs-mode 输入目录
│   └── {需求名称}.md                 # 用户需求文档
├── shots/                           # shot-mode 输入目录
│   └── {页面名称}.png               # 用户上传的截图
├── output/                          # 产出目录
│   ├── css/common.css               # CSS Token + 布局 + 全组件样式
│   ├── js/framework.js             # 框架注入脚本
│   ├── js/common.js                # 导航菜单 + 面包屑渲染
│   └── pages/*.html                # 各页面 HTML（只含主体内容）
└── prototype.config.json            # Skill 配置文件
```

### 2.2 Skill 文件结构

```
.claude/skills/prototype-generator/
├── SKILL.md                         # 入口文件
├── config.md                        # 配置管理指令
├── docs-mode/
│   ├── prompt-builder.md            # docs-mode 提示词构造器
│   └── template.md                  # 泛化 /goal 提示词模板
├── shot-mode/
│   ├── prompt-builder.md            # shot-mode 提示词构造器
│   └── vision-analyzer.md           # Qwen3-VL 截图分析指令
└── verifier.md                      # 产出验证指令
```

### 2.3 命令设计

| 命令 | 作用 |
|------|------|
| `/prototype:docs` | 文档驱动模式：需求 + 规范 → 原型 |
| `/prototype:shot` | 截图驱动模式：截图 + 规范 → 原型 |
| `/prototype:config` | 查看/修改配置（视觉模型、默认规范等） |

---

## 3. 外部 Skill 集成

构造提示词阶段联动 3 个外部 skill，验证阶段使用 Playwright MCP + 视觉模型。

### 3.1 集成顺序

**docs-mode**：
```
输入文档 → ui-ux-pro-max → frontend-design → 构造 /goal 提示词
```

**shot-mode**（截图语义化提前，再走设计系统）：
```
截图 → 视觉模型分析 → screenshot-to-code → ui-ux-pro-max → frontend-design → 构造 /goal 提示词
```

### 3.2 各 Skill 职责

```
    ┌──────────────────────────────────┐
    │ ui-ux-pro-max --design-system    │
    │ ────────────────────────         │
    │ 产出:                            │
    │  · 色彩方案（从规范提取/推荐）    │
    │  · 字体配对                      │
    │  · 组件推荐（表格/表单/弹窗）     │
    │  · UX 检查清单（§1-10）           │
    │  · 图表配色                      │
    └────────────┬─────────────────────┘
                 │ 设计系统 JSON
    ┌────────────▼─────────────────────┐
    │ frontend-design                  │
    │ ────────────────────────         │
    │ 审查设计系统:                     │
    │  · 是否落入 AI 三大陷阱？         │
    │    - 暖奶油底                    │
    │    - 极暗底+单亮色               │
    │    - 报纸风                      │
    │  · 排版是否有性格？              │
    │  · 记忆点（signature）是什么？    │
    │  · 香奈儿原则：还能拿掉什么？     │
    └────────────┬─────────────────────┘
                 │ 修正后的设计系统
    ┌────────────▼─────────────────────┐
    │ [仅 shot-mode]                   │
    │ 视觉模型 + screenshot-to-code     │
    │ ────────────────────────         │
    │ 截图 → Qwen3-VL 像素分析         │
    │  → screenshot-to-code 语义化     │
    │ 产出:                            │
    │  · 组件层级结构                   │
    │  · 颜色值（hex）/ 字号 / 字重     │
    │  · 间距 / 元素位置 / 尺寸         │
    │  · 交互元素清单                   │
    └────────────┬─────────────────────┘
                 │
    ┌────────────▼─────────────────────┐
    │ 构造 /goal 提示词                 │
    │ 合并:                            │
    │  · 需求文档 / 截图分析            │
    │  · CONSTRAINTS（默认+规范覆盖）   │
    │  · 设计系统 → 注入 CONTEXT        │
    │  · UX 清单 → 注入 VERIFY          │
    │  · 截图分析 → 注入页面清单        │
    └──────────────────────────────────┘
```

### 3.3 验证阶段的工具

| 工具 | 用途 |
|------|------|
| **Playwright MCP** | 自动打开生成页面，在 1366/1440/1920 分辨率截图 |
| **视觉模型（可配置）** | 对比截图与规范要求，标注偏差 |
| **agentcloak** | 可选替代 Playwright，适用于需要登录态的页面 |

---

## 4. 提示词模板设计

沿用 deepal `/goal` 模板结构，将段按可变性分为四层。

### 4.1 四层划分

#### 不可变层（原文保留）

| 段 | 理由 |
|----|------|
| **STOP RULES** | 4 条通用安全边界，与项目无关 |
| **OUTPUT** | 4 条通用交付标准 |
| **VERIFY**（主体） | 7 项通用原型 QA 流程 |

#### 主体不可变，值变量化层

**CONSTRAINTS** 的 13 条规则主体固定，以下标注值从规范文档提取覆盖：

| # | 约束 | 来源 |
|----|------|------|
| 1 | 不得实现后端 API 调用或真实业务逻辑 | 固定 |
| 2 | 使用模拟数据，尽量真实 | 固定 |
| 3 | 必须遵循 `{规范路径}` 的 CSS Token 和组件约束 | 变量化 |
| 4 | CSS 必须分离为独立文件 | 固定 |
| 5 | 每个页面对应一个独立的 HTML 文件 | 固定 |
| 6 | 必须使用 CSS 变量定义颜色、尺寸、间距，不得散落硬编码 | 固定 |
| 7 | 控件高度符合 `{从规范提取}`，间距落在 `{从规范提取}` 步进体系 | 变量化 |
| 8 | 表格数值列、金额列、百分比列右对齐 | 固定 |
| 9 | 日期格式 `{从规范提取}`，日期时间格式 `{从规范提取}` | 变量化 |
| 10 | 金额使用千分位，百分比保留 2 位小数 | 固定 |
| 11 | 表单需要基础交互效果 | 固定 |
| 12 | 菜单/导航点击必须能够跳转到对应页面 | 固定 |
| 13 | 状态使用 Tag 展示 | 固定 |

**默认值**（用户未提供规范时使用）：

```
- 控件高度: 24/32/40px
- 间距步进: 4px
- 日期格式: yyyy-mm-dd
- 日期时间格式: yyyy-mm-dd HH:mm:ss
```

#### 结构保持层

| 段 | 策略 |
|----|------|
| **PLAN** | 八步骨架固定，第一步参考路径变量化，第五步页面清单从需求提取 |
| **DONE WHEN** | 完成条件结构固定，页面数量、规范名变量化 |
| **PRIORITY** | 五步骨架固定，第 3 步 P0-P3 分级从需求提取 |

#### 内容驱动层

| 段 | 策略 |
|----|------|
| **CONTEXT** | 结构保留，全部内容从输入文档提取 |
| **页面清单** | 从需求文档自动提取 |
| **跳转关系** | 从需求文档自动提取 |

---

## 5. 两种模式的执行流程

### 5.0 依赖检查（两种模式共用，Step 0）

在进入具体模式前，先检查所需外部 Skill 是否已安装：

```
Step 0 — 依赖检查
  ├── 检查 ui-ux-pro-max skill 是否可用
  ├── 检查 frontend-design skill 是否可用
  ├── 检查 screenshot-to-code skill 是否可用（仅 shot-mode 必须）
  └── 任一缺失 → 提示安装命令并停止：
       "缺少依赖 Skill：ui-ux-pro-max、frontend-design。
        安装方法：
        - ui-ux-pro-max: 在 Claude Code 中运行 /install ui-ux-pro-max
        - frontend-design: 在 Claude Code 中运行 /install frontend-design
        - screenshot-to-code: 在 Claude Code 中运行 /install screenshot-to-code
        安装后重新运行 /prototype:docs 或 /prototype:shot"
```

### 5.1 docs-mode（文档驱动）

```
用户执行 /prototype:docs

Step 1 — 输入检查
  ├── 检查 prototype/ 目录是否存在
  │    否 → 自动创建目录结构，提示用户放入 规范.md 和需求文档
  ├── 检查 prototype/规范.md 是否存在
  │    否 → 提示用户放入规范文档后重试
  └── 检查 prototype/docs/ 下是否有 .md 文件
       否 → 提示用户放入需求文档后重试

Step 2 — 文档解析
  ├── 解析 规范.md:
  │    · 提取 CSS Token（颜色、字号、间距、圆角、阴影）
  │    · 提取组件规范（表格、表单、弹窗、抽屉、标签等）
  │    · 提取检查清单
  │    · 提取品牌色/Logo
  └── 解析需求.md:
       · 提取页面清单（一级/二级导航）
       · 提取页面跳转关系
       · 提取字段列表、操作流
       · 提取状态流转

Step 3 — 设计系统生成（外部 Skill）
  ├── 调用 ui-ux-pro-max --design-system
  │    输入：规范文档提取值 + 产品类型关键词
  │    产出：色彩方案、字体配对、组件推荐、UX 清单
  └── 调用 frontend-design 审查
       审视设计系统的独特性、避开 AI 陷阱

Step 4 — 构造 /goal 提示词
  ├── 填充不可变层（原文）
  ├── CONSTRAINTS 默认值 + 规范覆盖
  ├── 结构保持层按文档内容填充
  ├── 注入设计系统到 CONTEXT
  └── 输出完整提示词并展示给用户确认

Step 5 — 执行生成
  └── Claude 执行 /goal 提示词 → 生成 prototype/output/ 下所有文件

Step 6 — 自动化验证
  ├── Playwright 打开每个页面
  │    ─ 1366×768 截图
  │    ─ 1440×900 截图
  │    ─ 1920×1080 截图
  ├── 检查项：
  │    · CSS 变量正确引用
  │    · 控件高度符合规范
  │    · 表格数值列右对齐
  │    · 所有导航链接可跳转
  │    · 无溢出/遮挡
  └── 输出验证报告（通过/未通过/偏差标注）

Step 7 — 输出
  ├── 文件清单
  ├── 页面跳转关系说明
  └── 未实现交互功能列表
```

### 5.2 shot-mode（截图驱动）

```
用户执行 /prototype:shot

Step 1 — 输入检查
  ├── 检查 prototype/ 目录
  ├── 检查 prototype/规范.md（可选但推荐）
  └── 检查 prototype/shots/ 下是否有截图
       否 → 提示用户放入截图后重试

Step 2 — 视觉模型配置（首次使用）
  └── 未配置 → 引导选择 provider + API Key → 写入 prototype.config.json

Step 3 — 像素级分析（视觉模型）
  ├── 逐张截图调用视觉模型（默认 Qwen3-VL-Plus）
  └── 输出结构化描述：
       · 布局结构（header / sidebar / content / footer）
       · 组件层级（table > thead > tr > th...）
       · 颜色值（hex）/ 字号（px）/ 字重
       · 间距（px）/ 元素位置（x, y）/ 尺寸（w, h）
       · 交互元素（按钮、输入框、下拉、标签）

Step 4 — 截图语义化（screenshot-to-code）
  ├── 调用 screenshot-to-code skill
  └── 将像素描述转为组件语义描述：
       · "220×32px 蓝色矩形 #1677ff" → "32px 主按钮 primary"
       · "左侧 220px 深色区域" → "侧边导航 sidebar"

Step 5 — 设计系统生成（同 docs-mode Step 3）
  ├── ui-ux-pro-max --design-system
  └── frontend-design 审查

Step 6 — 构造 /goal 提示词
  ├── 合并：截图分析 + 设计系统 + CONSTRAINTS
  ├── 发现截图与规范的差异 → 标注冲突 → 用户决定优先级
  └── 展示提示词确认

Step 7 — 执行生成（同 docs-mode）

Step 8 — 像素级验证
  ├── 截图与生成页面逐区对比
  ├── 标注偏差（颜色差异、间距偏移、字号偏差）
  ├── 判定标准：
  │    · S1-S4 项偏差超出阈值 → 记录为「需修正」
  │    · 需修正项 > 3 或存在主色/布局级偏差 → 自动重生成
  │    · ≤ 3 项且均为微调级 → 展示报告让用户决定
  └── 用户确认是否重生成（通常 1-2 轮迭代）
```

---

## 6. 配置管理

### 6.1 配置文件

`prototype/prototype.config.json`（首次使用自动生成）：

```json
{
  "vision_model": {
    "provider": "qwen3-vl-plus",
    "api_key": "",
    "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
    "model_name": "qwen3-vl-plus"
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

### 6.2 配置命令 `/prototype:config`

```
/prototype:config              → 查看当前配置
/prototype:config vision       → 修改视觉模型配置
/prototype:config spec         → 修改默认规范路径
/prototype:config verify off   → 关闭自动验证
```

### 6.3 支持的视觉模型提供商

| Provider | 默认模型 | 需要 API Key |
|----------|----------|:---:|
| qwen3-vl-plus | qwen3-vl-plus | 是 |
| claude-vision | claude-sonnet-4-6 | 否（内置） |
| gpt-4o | gpt-4o | 是 |

---

## 7. 验证规范

### 7.1 自动化检查项

| # | 检查项 | 方法 |
|----|--------|------|
| 1 | CSS 变量定义且被正确引用 | 扫描 CSS |
| 2 | 字体使用系统字体栈 | 检查 font-family |
| 3 | 数字设置 tabular-nums | 检查 font-variant-numeric |
| 4 | 控件高度符合 24/32/40px | DOM 测量 |
| 5 | 间距落在 4px 步进体系 | DOM 测量 |
| 6 | 表格数值列右对齐 | 截图分析 |
| 7 | 日期/金额格式正确 | 截图 + 文本分析 |
| 8 | 所有导航链接可跳转 | Playwright 逐链点击 |
| 9 | 1366/1440/1920 分辨率无溢出 | 多分辨率截图 |
| 10 | 空/加载/错误/禁用状态有样式 | 手动补充 |
| 11 | 弹窗/抽屉有圆角+阴影+关闭路径 | 截图分析 |
| 12 | 色彩使用规范色值 | CSS Token 扫描 |

### 7.2 shot-mode 额外检查

| # | 检查项 | 方法 |
|----|--------|------|
| S1 | 主色偏差 < 5%（ΔE 色差） | 视觉模型对比 |
| S2 | 组件层级与截图一致 | 结构对比 |
| S3 | 字号偏差 < 1px | 视觉模型测量 |
| S4 | 间距偏差 < 2px | 视觉模型测量 |

---

## 8. 错误与边界处理

| 场景 | 处理 |
|------|------|
| 外部 Skill 未安装 | 停止，列出缺失的 Skill 和安装命令，安装后重试 |
| 规范文档无法解析 | 停止，列出无法解析的部分，请用户补充 |
| 需求文档页面清单不明确 | 停止，列出已识别的页面，请用户确认 |
| 截图质量过低 | 提示用户提供更高清截图（建议 2x 分辨率） |
| 视觉模型 API 调用失败 | 回退到仅依赖规范文档，不做像素级分析 |
| 截图与规范冲突 | 标注冲突项，让用户决定优先截图还是规范 |
| 生成页面数 > 50 | 分批次生成，每批 ≤ 15 页 |
| Playwright 不可用 | 回退到手动验证清单 |

---

## 9. 与 deepal 原工作流的对应关系

| 原工作流 | 新 Skill |
|----------|----------|
| 手动准备需求.md | 自动检查 prototype/docs/，缺则提示 |
| 手动准备 UI 规范 | 自动检查 prototype/规范.md，缺则使用内置默认 |
| 手动修改 /goal 提示词 | 自动构造，填充四层模板 |
| 手动执行 /goal | 自动执行 |
| 手动点链接验证跳转 | Playwright 自动化 |
| 手动检查样式 | Playwright 截图 + 视觉模型对比 |
| 人工判断 | 同左——不可替代的设计决策保留人工确认 |

---

## 10. 后续迭代方向

- [ ] 支持 React/Vue 组件化产出（不只是纯 HTML）
- [ ] 支持 Axure RP 原生 .rp 格式导出
- [ ] 批量截图模式：输入文件夹批量处理
- [ ] 规范文档自动从 PDF 提取（复用 pdfplumber）
- [ ] 设计系统跨项目持久化复用
