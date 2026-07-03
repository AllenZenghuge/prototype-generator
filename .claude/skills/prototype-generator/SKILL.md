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
| 2026-07-03 | vision-analyzer 新增 Phase 0（上下文收集+确认），修复多截图列漏识别；规范.md 新增 §14 标准页面模板；prompt-builder 新增生成后自检；组件类型枚举细化（button-icon/text/link）；质量审计完成 |

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
4. Read `shot-mode/vision-analyzer.md` and follow its pixel analysis instructions
5. Read `shot-mode/prompt-builder.md` and follow its instructions exactly
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
| Vision model API failure (shot-mode) | Fall back to spec-only generation, skip pixel analysis |
| Screenshot conflicts with spec | List conflicts, ask user which takes priority |
| >50 pages detected | Split into batches of ≤15 pages |
| Playwright unavailable | Fall back to manual verification checklist |
