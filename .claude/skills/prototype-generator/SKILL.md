---
name: prototype-generator
description: Generate interactive HTML prototypes from requirements docs + design specs or from screenshots. Output compatible with Axure/墨刀 import. Usage: /prototype-generator [docs|shot|config]
---

# Prototype Generator

根据需求文档或 UI 截图，生成可交互的 HTML 原型页面，兼容 Axure/墨刀导入。

## Changelog

| 日期 | 变更内容 |
|------|---------|
| 2026-06-30 | v1.0 初始版本，支持 docs/shot/config 三种模式，参数路由 |
| 2026-07-01 | 命令路由修复（/prototype:docs→/prototype-generator docs）、shot-mode 端到端测试通过、发布 GitHub |
| 2026-07-20 | v1.1 全面优化 — Vision API 可靠性（脚本化执行+超时重试+失败不跳过）；页面隔离铁律（共享文件修改规则+菜单隔离）；强制自检 Phase 7（JS语法/多页回归/§14模板/CSS Token/控件高度）；子组件专用 prompt（弹窗/侧边栏）；5种截图关系枚举；基线自动管理；分析结果强制保存 |

## How to Use

本 skill 通过参数选择模式，调用方式：

| 调用 | 作用 |
|------|------|
| `/prototype-generator config` | 查看/修改配置（首次使用必须先运行） |
| `/prototype-generator docs` | 需求文档 + 设计规范 → HTML 原型 |
| `/prototype-generator shot` | UI 截图 → 像素级 HTML 原型 |
| `/prototype-generator` | 显示此菜单，引导选择模式 |

**调用方式有两种：**
- 斜杠命令：`/prototype-generator docs`
- Skill 工具：`Skill("prototype-generator", {args: "docs"})`

## Mode Router

读取 `args` 参数（即 `/prototype-generator` 后面的第一个词），按以下逻辑路由：

```
args 为空 → 显示模式选择菜单，询问用户：
  "选择一个模式：
   1. docs — 根据需求文档 + 规范文档生成原型
   2. shot — 根据 UI 截图生成像素级原型
   3. config — 查看/修改配置
   输入数字或名称："

args = "config" → 跳到 §Config Mode
args = "docs"   → 运行 Step 0 依赖检查，通过后跳到 §Docs Mode
args = "shot"   → 运行 Step 0 依赖检查，通过后跳到 §Shot Mode
其他值         → 提示 "未知模式: {args}。可用: docs, shot, config"
```

## Step 0 — Dependency Check (docs/shot 模式前执行)

在执行 docs 或 shot 模式前，检查所需外部 skill 是否已安装：

```
必需 skill:
  - ui-ux-pro-max      → 设计系统生成
  - frontend-design    → 设计审查和独特性检查
  - screenshot-to-code → [仅 shot 模式] 截图语义分析

检查方法:
  尝试通过 Skill 工具调用每个 skill。如果返回 "not found" 则缺失。

任一缺失则停止并输出:
  "缺少依赖 Skill：{skill_names}。
   安装方法：
   - ui-ux-pro-max: 在 Claude Code 中运行 /install ui-ux-pro-max
   - frontend-design: 在 Claude Code 中运行 /install frontend-design
   - screenshot-to-code: 在 Claude Code 中运行 /install screenshot-to-code
   安装后重新运行 /prototype-generator [docs|shot]"
```

省略 screenshot-to-code 不会阻止 docs 模式。

## Docs Mode

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

## Shot Mode

**Input required:**
- `prototype/规范.md` — UI design spec (optional but recommended)
- `prototype/shots/*.png` — Screenshots (required)

**Execution flow:**

1. Read `config.md` to learn configuration and check vision model setup
2. Verify `prototype/` directory; create if missing
3. Verify `prototype/shots/` has screenshots; if not, prompt user
4. Read `shot-mode/vision-analyzer.md` and follow its pixel analysis instructions:
   - **Phase 0.3**: 确认截图关系类型（5 种枚举：独立/同页不同态/水平分割/垂直分割/主从）
   - **Phase 0.5**: [表单页] 提取侧边栏菜单结构
   - **Phase 1**: API 调用（脚本化执行 + 超时重试 + 失败提示用户决策）
   - **Phase 2**: 强制保存分析结果（每次 API 调用后立即保存，含失败时的 error.json）
5. Read `shot-mode/prompt-builder.md` and follow its instructions exactly:
   - **Phase 0.5**: 遵守页面隔离铁律（framework.js/common.js/common.css 修改规则）
   - **Phase 0.6**: 基线检查（已有页面自动创建基线备份）
   - **Phase 1-6**: 生成 HTML 原型
   - **Phase 7**: 强制自检（JS 语法 + 多页回归 + §14 模板 + CSS Token + 控件高度）
6. Read `docs-mode/template.md` for the /goal prompt template structure
7. After generation completes, read `verifier.md` and follow its instructions

## Config Mode

Read `config.md` and follow its instructions for the requested config operation.

## Error Handling

See `config.md` for configuration errors. For execution errors:

| Scenario | Action |
|----------|--------|
| Missing input files | Prompt user with exact paths expected |
| Spec doc unparseable | Stop, list sections that couldn't be parsed |
| Requirements have unclear page list | Stop, list detected pages, ask user to confirm |
| Screenshot quality too low | Prompt user for higher resolution (2x recommended) |
| Vision model API failure (shot-mode) | Save error file, prompt user with 3 options: retry / switch model / manual description. DO NOT skip. |
| Screenshot conflicts with spec | List conflicts, ask user which takes priority |
| >50 pages detected | Split into batches of ≤15 pages |
| Playwright unavailable | Fall back to manual verification checklist |
