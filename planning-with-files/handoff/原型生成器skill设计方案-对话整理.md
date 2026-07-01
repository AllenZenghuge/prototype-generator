---
title: 原型生成器 Skill 设计方案 — 对话整理
date: 2026-07-01
tags:
  - prototype
  - skill
  - html-css
  - design-system
  - handoff
  - github
aliases:
  - deepal 原型工作流分析与 skill 设计
  - '/prototype-generator docs shot config'
description: >
  基于 deepal 项目已验证的 HTML 原型生成工作流分析，与 Allen 逐步对齐需求后，
  完成了通用型 prototype-generator skill 的完整设计、实现、测试和 GitHub 发布。
  Skill 支持 docs-mode（需求文档→原型）和 shot-mode（截图→原型），
  集成 ui-ux-pro-max、frontend-design、screenshot-to-code 三个外部 skill，
  产出兼容 Axure/墨刀导入。已通过长安汽车销项开票截图的端到端测试。
source: Claude Code 对话
---

# 原型生成器 Skill 设计方案 — 对话整理

## 修订记录

| 时间 | 修订内容 |
|------|---------|
| 2026-06-30 14:52 | 初始版本，整理对话中所有设计决策和产出 |
| 2026-07-01 09:24 | 更新：实现完成、shot-mode 测试通过、GitHub 发布、命令路由修复、Changelog 机制 |

---

## 1. 对话目标

- 分析 `/个人资料/虹信/项目/01深蓝/deepal` 目录的完整工作流
- 基于分析结果，设计一个通用型 Claude Code Skill
- 能力：(1) 需求文档 + 设计规范 → 原型；(2) 截图 → 像素级原型

---

## 2. Deepal 工作流分析结果

### 2.1 输入文件

| 文件 | 作用 |
|------|------|
| `goal 提示词模板（HTML 原型开发）.md` | /goal 模板：CONTEXT → 页面清单 → CONSTRAINTS → PRIORITY → PLAN → DONE WHEN → VERIFY → OUTPUT → STOP RULES |
| `前端页面需求说明.md` | PRD，含 9 个一级导航、40+ 页面、字段/流程/校验规则 |
| `虹信PC端UI规范3.0_HTML约束文档.md` | 107 页 UI 规范提炼：CSS Token、组件约束、检查清单 |
| `header.png` / `new.png` | 参考截图 |
| `收票台账-可抵扣字段逻辑设计.md` | 业务逻辑设计文档（另一场景） |

### 2.2 产出文件

| 文件 | 说明 |
|------|------|
| `css/common.css` | 1470 行：CSS Token + 布局 + 全组件（表格/表单/弹窗/抽屉/Tabs/Tag/Timeline/Steps...） |
| `js/framework.js` | 框架注入脚本，把页面内容包装到统一布局（header + sidebar + content） |
| `js/common.js` | 导航菜单数据 + 侧边栏渲染 + 面包屑渲染 |
| `pages/*.html` | 40 个独立页面文件，只含主体内容，不含布局壳 |

### 2.3 关键模式

- **框架注入**：每个页面只有内容 HTML，`framework.js` 在页面加载后注入统一布局
- **CSS 分层**：公共样式独立文件 + 页面专属内联 `<style>`
- **三层 JS 架构**：framework.js（布局注入）→ common.js（导航/面包屑）→ 页面内联脚本

### 2.4 验证工具链

| 工具 | 用途 |
|------|------|
| Playwright MCP | 自动截图、多分辨率检查 |
| MiniMax understand_image | 视觉对比 |
| 手动点击验证 | 导航跳转确认 |

### 2.5 无关内容

`.venv`（pdfplumber 读规范 PDF）、`.playwright-mcp`（截图验证日志）、`.claude/skills/prototype`（开发中临时原型探索，不同目的）

---

## 3. 需求对齐过程

### 3.1 逐轮确认的关键决策

| # | 决策点 | 结果 |
|----|--------|------|
| 1 | Skill 适用范围 | **通用型**，不限于虹信/深蓝 |
| 2 | 两个能力的关系 | **一个 Skill，两个子命令** |
| 3 | 产出兼容性 | 导入 **Axure/墨刀**，要求 HTML 结构干净 |
| 4 | 截图视觉模型 | **Qwen3-VL + Claude 两阶段**，模型可配置 |
| 5 | 产出文件结构 | 沿用 deepal 结构，**品牌变量由规范驱动** |
| 6 | 设计方案选择 | **方案 A**：模板驱动的生成系统 |
| 7 | 项目目录 | `prototype/` 根目录，`规范.md` 放顶层，输入分 `docs/` `shots/` 子目录 |
| 8 | 提示词模板分层 | 四层：不可变层 / 值变量化层 / 结构保持层 / 内容驱动层 |
| 9 | CONSTRAINTS 默认值 | 使用 deepal goal 模板中的原值，用户规范可覆盖 |
| 10 | 视觉模型配置 | 首次使用引导配置，`/prototype:config` 可修改 |
| 11 | 三 Skill 联动 | ui-ux-pro-max → frontend-design → 构造提示词；shot-mode 加 screenshot-to-code |
| 12 | Playwright vs agentcloak | 默认 Playwright，agentcloak 做可选替代（登录态页面） |
| 13 | 依赖检查 | Step 0 先检查外部 Skill 是否安装，缺失则提示安装命令 |

### 3.2 提示词模板四层划分

| 层 | 包含段 | 策略 |
|----|--------|------|
| **不可变层** | STOP RULES、OUTPUT、VERIFY（除分辨率值） | 原文保留 |
| **主体不可变，值变量化** | CONSTRAINTS | 13 条固定，4 条从规范注入（控件高度、间距步进、日期格式） |
| **结构保持层** | PLAN、DONE WHEN、PRIORITY | 骨架固定，数量/名称/路径变量化 |
| **内容驱动层** | CONTEXT、页面清单、跳转关系 | 从输入文档提取 |

---

## 4. 最终设计方案

### 4.1 命令

| 命令 | 作用 |
|------|------|
| `/prototype-generator docs` | 文档驱动：需求.md + 规范.md → HTML 原型 |
| `/prototype-generator shot` | 截图驱动：截图.png + 规范.md → HTML 原型 |
| `/prototype-generator config` | 配置管理 |

> 注：原设计为 `/prototype:docs`，实现后发现 Claude Code 不支持子命令路由，
> 改为通过 Skill 参数路由 `/prototype-generator [docs|shot|config]`。

### 4.2 目录结构

```
{项目根目录}/prototype/
├── 规范.md                          # UI 规范（docs-mode 必选 / shot-mode 可选）
├── docs/                            # docs-mode 输入
├── shots/                           # shot-mode 输入
├── output/                          # 产出
│   ├── css/common.css
│   ├── js/framework.js
│   ├── js/common.js
│   └── pages/*.html
└── prototype.config.json            # 配置（视觉模型等）
```

### 4.3 外部 Skill 集成顺序

- **docs-mode**：输入文档 → ui-ux-pro-max → frontend-design → 构造提示词
- **shot-mode**：截图 → 视觉模型 → screenshot-to-code → ui-ux-pro-max → frontend-design → 构造提示词

### 4.4 执行流程

两种模式共用 Step 0（依赖检查），然后各自 7-8 步：输入检查 → 文档解析/像素分析 → 设计系统 → 构造提示词 → 执行 → 验证 → 输出

---

## 5. 产出物

| 文件 | 路径 |
|------|------|
| **设计 Spec** | `docs/superpowers/specs/20260630-prototype-generator-skill-design.md` |
| **实现计划** | `docs/superpowers/plans/20260630-prototype-generator-skill-implementation.md` |
| **Skill 文件 (7)** | `.claude/skills/prototype-generator/` |
| **GitHub** | https://github.com/AllenZenghuge/prototype-generator |

### 5.1 Skill 文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `SKILL.md` | 89 | 入口 + 参数路由 + Step 0 依赖检查 + Changelog |
| `config.md` | 77 | 配置管理 + 首次使用向导 |
| `docs-mode/template.md` | 145 | 四层 /goal 提示词模板 |
| `docs-mode/prompt-builder.md` | 161 | 文档解析 → ui-ux-pro-max → frontend-design → 填模板 |
| `shot-mode/vision-analyzer.md` | 132 | Qwen3-VL API 调用 + JSON Schema |
| `shot-mode/prompt-builder.md` | 81 | 截图分析 → screenshot-to-code → 填模板 |
| `verifier.md` | 162 | 12 项 docs + 4 项 shot 自动化验证 |

---

## 6. 实现与测试

### 6.1 实现过程发现的修正

| # | 问题 | 修正 |
|----|------|------|
| 1 | `/prototype:docs` 等子命令无法识别 | 改为参数路由 `/prototype-generator docs` |
| 2 | SKILL.md 缺少版本记录 | 新增 Changelog 章节 |
| 3 | `.claude/skills/` 写入触发安全告警 | 正常——skill 创建即 self-modification |
| 4 | cc_test 其他文件出现在源代码管理 | .gitignore 排除非 skill 文件 |
| 5 | 虹信规范文档不能推公开仓库 | .gitignore 排除 `prototype/` 全部 |

### 6.2 Shot-Mode 端到端测试

**测试截图**：长安汽车销项开票 — 所有单据（列表页，空数据状态）

**Pipeline 执行**：
```
Step 0  ✅ 依赖检查 (3/3)
Phase 1 ✅ Qwen3-VL-Plus 像素分析
Phase 2 ✅ screenshot-to-code 语义映射
Phase 3 ✅ ui-ux-pro-max 设计系统
Phase 4 ✅ frontend-design 审查
Phase 5 ✅ 生成输出 (589行 CSS + 127行 HTML + JS)
```

**产出质量**：筛选栏、Tab、表格 9 列、空状态、分页均已还原，主色 #3B82F6 与截图一致。

### 6.3 配置状态

```
Vision model:   Qwen3-VL-Plus
Default spec:   ./规范.md
Verification:   enabled (playwright)
```

---

## 7. 下一步

Skill 已完成设计、实现、测试、发布。后续使用中迭代：
- docs-mode 端到端测试（需准备需求文档）
- Playwright 自动验证集成测试
- 规范文档 PDF 自动提取（复用 pdfplumber）
- CHANGELOG 按迭代持续更新
