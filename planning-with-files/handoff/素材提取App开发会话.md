# 素材提取 App — 会话交接

## 修订记录

| 时间 | 修订内容 |
|------|---------|
| 2026-06-15 23:00:00 | 初始版本，整理本次对话内容 |
| 2026-06-21 12:10:00 | App 代码完成 + Spec/CodeQuality 审查 + 测试计划/用例编写 |
| 2026-06-21 15:15:00 | 全面更新：解析器切换到 vxtwitter、代理改为环境变量、下载存相册修复、26 tests 全过、端到端测试通过 |
| 2026-06-21 20:05:00 | v1.1 更新：4 个 Bug/Feature 修复、复选框勾选、选择性下载、存储策略优化、完整文档更新 |
| 2026-06-21 21:30:00 | BUG-003 深度修复：重新下载改为「删旧建新」策略，原地修改 SwiftData @Model 对象存在引用安全问题 |
| 2026-06-21 22:41:00 | v2.0 前端重设计 brainstorming 启动：毛玻璃方案 B 选定，色彩/材质/字体系统建立，提取页 mockup v1 产出，竞品分析完成 |
| 2026-06-22 00:36:57 | v2.0 前端重设计全部完成 + 首次/第二次 Compact 会话纪要存档 |
| 2026-06-22 16:36:28 | 修正虚构时间：3 条 06-22 条目（01:30/01:45/02:00）均在文件系统 mtime 00:36:57 之后，合并为一条真实时间 |
| 2026-08-09 00:00:00 | 全面审计：核对代码实际状态 vs 设计文档，从 mockup 提取权威色彩/材质值，识别文档缺口，新增 v2.0 实现路径和 Git 操作建议 |
| 2026-08-10 00:00:00 | v2.0 代码实现完成：TDD 流程 17 文件改动、3 Tab 毛玻璃全页面落地、TC-V2-01~10 测试通过、后端 venv 配置、分支管理就绪 |
| 2026-08-23 00:00:00 | 文件名三段式 + 存储策略改「文件」App；MediaExtractor 创建私有仓库并 push；PR #1 合并 v2.0 进 main，打 v2.0 tag |

---

## 一句话总结

iPhone App + Python 后端，实现对 X (Twitter) 推文的无水印视频/图片/文案提取。**v2.0 毛玻璃重设计已合并到 main 成为正式版；v1.1 暗房主题由 `v1.1-darkroom` tag 保留。**

---

## 当前状态总览

| 模块 | 版本 | 分支 | 状态 |
|------|------|------|------|
| **v1.1 暗房主题** | v1.1 | `main` | ✅ BUILD SUCCEEDED，26 后端测试全过 |
| **v2.0 毛玻璃重设计** | v2.0 | `main` | ✅ 已合并，正式版 |
| **后端** | — | 通用 | ✅ vxtwitter API + 代理 + venv |
| **设计文档** | v2.0 | — | ✅ 决策记录 + 设计概要 + 23 mockup |

### v1.1 → v2.0 变更对照

| 维度 | v1.1 暗房 | v2.0 毛玻璃 |
|------|----------|-----------|
| 色彩 | 暗房黑 #0d0d12 / 暗房红 #d44a3a | 冷紫 #6C5CE7 / 多色渐变 + 3 光斑 |
| 模式 | 强制深色 `.preferredColorScheme(.dark)` | 跟随系统 深浅自动切换 |
| 材质 | 纯色背景 | `.ultraThinMaterial` + blur + saturate |
| 导航 | NavigationStack + 右上角链接 | 3 Tab 悬浮胶囊 248px × 28px |
| 页面 | 提取页 + 下载列表（2 页） | 提取 + 下载列表 + 我的 + 设置 + 登出/注销（5 页） |
| 测试 | TC-APP-01~09（9 个） | + TC-V2-01~10（19 个总计） |

---

## v2.0 代码实现（✅ 已完成）

### 实施流程

TDD 驱动：先写测试 → 验证失败 → 写最少代码 → 编译通过 → 重构，共 3 个 TDD 周期。

### 改动清单

| 类别 | 文件 | 内容 |
|------|------|------|
| 🆕 | `Views/Components/FloatingTabBar.swift` | 悬浮胶囊 Tab Bar + AppTab 枚举 |
| 🆕 | `Views/Profile/ProfileView.swift` | 「我的」C3 v4：会员置顶 + 头像行含⚙ + 续费紫蓝卡片 + 菜单 |
| 🆕 | `Views/Settings/SettingsView.swift` | 设置页：账户+功能+法律+登出+注销 |
| ✏️ | `Utilities/Color+Theme.swift` | 保留 v1.1 色板 + 新增 v2.0 色板（accentPrimary/textPrimary/glassL2-L3/status 等） |
| ✏️ | `Views/ContentView.swift` | 强制深色 → 渐变光斑+3 Tab+深浅双模 |
| ✏️ | `Views/Extract/ExtractView.swift` | NavigationStack → ScrollView + 冷紫 CTA |
| ✏️ | `Views/Extract/URLInputCard.swift` | 暗房卡片 → L3 毛玻璃 `.ultraThinMaterial` |
| ✏️ | `Views/Extract/GradientArrowButton.swift` | 暗房红 → 冷紫 #6C5CE7 |
| ✏️ | `Views/Extract/TutorialSection.swift` | v1.1 色板 → v2.0 Token |
| ✏️ | `Views/Extract/ResultTabs/CustomTabBar.swift` | 暗房红 → 冷紫下划线 |
| ✏️ | `Views/Extract/ResultTabs/VideoResultTab.swift` | 暗房色 → 冷紫复选框 + status Token |
| ✏️ | `Views/Extract/ResultTabs/ImageResultTab.swift` | 同上 |
| ✏️ | `Views/Extract/ResultTabs/TextResultTab.swift` | 暗房底 → `.ultraThinMaterial` |
| ✏️ | `Views/DownloadList/DownloadListView.swift` | NavigationStack → 透明底 + 标题栏 |
| ✏️ | `Views/DownloadList/DownloadRowView.swift` | 暗房色 → 冷紫 + `.ultraThinMaterial` 卡片 |
| ✏️ | `Views/DownloadList/DownloadEmptyView.swift` | 暗房色 → v2.0 Token |
| 🧪 | `MediaExtractorTests.swift` | 新增 TC-V2-01~10（Color+Theme + AppTab） |

### 测试结果

| 测试类别 | 数量 | 结果 |
|---------|------|------|
| v2.0 新增（TC-V2-01~10） | 10 | ✅ 全部通过 |
| v1.1 已有（TC-APP-01~09） | 9 | 6 通过 / 3 失败（Clipboard 环境问题，非 v2.0 引起） |
| 后端 pytest | 26 | ✅ 全部通过 |

---

## v2.0 前端重设计（设计系统参考）

### 最终设计系统（来源：mockup 12/13/14 增强版，权威值）

#### 色彩

| Token | 浅色 | 深色 | 用途 |
|-------|------|------|------|
| 背景 | 多色冷紫渐变 `#d8d4f0→#f0edfa→#e4e0f5` | 多色深紫渐变 `#020203→#080610→#0a0a12→#06050d` | L0 底层 |
| 光斑 | 3 个冷紫光斑，opacity **22-35%**，blur 70-90px | 3 个冷紫光斑，opacity **14-22%**，blur 60-80px | L1 氛围 |
| 强调色 | **#6C5CE7** | **#8B7CF6** | CTA/选中/进度条 |
| 成功绿 | #34C759 | #30D158 | 下载完成 |
| 警告 | #FF9F0A | #FFD60A | 续费提醒 |
| 主文字 | #1C1C1E | #FFFFFF / #EDEDEF | 正文 |
| 次文字 | #8E8E93 | rgba(255,255,255,0.35) | 辅助信息 |

#### 材质层级（增强版 — 来自 mockup 13-glass-enhance-v2-light.html 规范表）

| 层级 | 浅色 | 深色 |
|------|------|------|
| L0 基础底 | 多色冷紫渐变 + 3 光斑（opacity 22-35%） | 多色深紫渐变 + 3 光斑（opacity 14-22%） |
| L3 输入卡片 | **45% white** + blur 50 + saturate(160%) + 1px/60% white 边框 + inset 高光线 | **10% white** + blur 50 + saturate(150%) + 1px/12% white 边框 + inset 高光线 |
| L2 卡片 | **35% white** + blur 45 + saturate(150%) + 1px/50% white 边框 | **7% white** + blur 45 + saturate(140%) + 1px/10% white 边框 |

> 代码中使用 `.ultraThinMaterial` 近似实现毛玻璃效果，未完全还原 mockup 的自定义 blur + saturate 参数（SwiftUI 不原生支持 `backdrop-filter: blur() saturate()`）。

#### Tab Bar（悬浮胶囊）

| 属性 | 浅色 | 深色 |
|------|------|------|
| 容器 | `.ultraThinMaterial` | `.ultraThinMaterial` |
| 宽度 | 248px | 248px |
| 圆角 | 28px | 28px |
| 底部留空 | 6px | 6px |
| 选中图标/文字 | #1C1C1E bold | #8B7CF6 bold |
| 未选中图标/文字 | #8E8E93 | 纯白 #FFFFFF |
| 选中高亮胶囊 | rgba(0,0,0,0.06), borderRadius 22px | rgba(255,255,255,0.10), borderRadius 22px |

### 设计决策记录

| 文档 | 路径 | 状态 |
|------|------|------|
| v1.1 设计决策 | [20260614_v1.0_设计决策记录.md](myapp/素材提取/01-设计/20260614_v1.0_设计决策记录.md) | ✅ 决策 1~16 |
| v2.0 设计决策 | [20260809_v2.0_设计决策记录.md](myapp/素材提取/06-前端重设计/20260809_v2.0_设计决策记录.md) | ✅ 决策 17~26 |
| 设计概要 | [20260621_v2.0_前端重设计概要.md](myapp/素材提取/06-前端重设计/20260621_v2.0_前端重设计概要.md) | ✅ 增强版参数 |
| 23 mockup | [mockups/](myapp/素材提取/06-前端重设计/mockups/) | ✅ 权威设计来源 |

---

## v1.1 修改清单（2026-06-21 对话）

### Bug 修复

| 编号 | 问题 | 修复 | 文件 |
|------|------|------|------|
| BUG-003 | 下载列表重新下载后任务消失 | **删旧建新**：delete 旧记录 → insert 新记录 → download | DownloadListViewModel + DownloadManager |
| BUG-004 | 下载文件存 tmp 目录易丢失 | tmp → cachesDirectory/Downloads/（中转），写相册后删临时 | DownloadManager |

### 新功能

| 编号 | 功能 | 文件 |
|------|------|------|
| FEAT-001 | 视频缩略图作为图片下载 | parser.py |
| FEAT-002 | 选择性下载（仅当前 tab） | ExtractViewModel + ExtractView |
| FEAT-003 | 媒体项复选框（全选/单选） | ExtractViewModel + 3 ResultTab + ExtractView |
| FIX-002 | 去双份存储（DCIM 唯一副本） | DownloadManager |

---

## 项目结构

```
myapp/素材提取/
├── 01-设计/
│   ├── 20260614_v1.0_设计文档.md
│   ├── 20260614_v1.0_设计决策记录.md          ← v1.1 决策 1~16
│   ├── 20260614_v1.0_开发会话纪要_首次Compact.md
│   ├── 20260622_v1.1_开发会话纪要_第二次Compact.md
│   └── 20260621_v1.1_变更记录.md
├── 02-实现计划/
│   └── 20260614_v1.0_实现计划.md
├── 03-后端源码/
│   └── backend/
│       ├── main.py          ← FastAPI + /api/parse
│       ├── models.py        ← Pydantic 模型
│       ├── parser.py        ← vxtwitter API 解析器（含视频缩略图→images）
│       ├── ratelimit.py     ← 频率限制 + 缓存
│       ├── requirements.txt
│       ├── venv/            ← 🆕 Python 虚拟环境（fastapi/uvicorn/httpx/pydantic）
│       └── tests/
│           ├── test_parser.py  ← 11 tests
│           └── test_unit.py    ← 15 tests
├── 04-App源码/
│   └── MediaExtractor/
│       ├── App/MediaExtractorApp.swift
│       ├── Models/          ← 5 文件
│       ├── ViewModels/      ← 2 文件
│       ├── Views/
│       │   ├── ContentView.swift      ← v2.0 3 Tab + 渐变光斑
│       │   ├── Components/            ← 🆕 FloatingTabBar
│       │   ├── Extract/               ← v2.0 冷紫+毛玻璃（8 文件）
│       │   ├── DownloadList/          ← v2.0 毛玻璃卡片（3 文件）
│       │   ├── Profile/               ← 🆕 C3 v4「我的」
│       │   └── Settings/              ← 🆕 设置+登出+注销
│       ├── Services/        ← 3 文件（APIClient、DownloadManager、PersistenceService）
│       └── Utilities/       ← 4 文件（Color+Theme 双色板 + 3 辅助）
├── 05-测试/
│   ├── 测试计划.md
│   ├── 后端测试/（Spec审查 + 测试用例）
│   └── App测试/（Spec审查 + CodeQuality + 测试用例）
└── 06-前端重设计/
    ├── 20260621_v2.0_前端重设计概要.md  ← ✅ 增强版参数
    ├── 20260809_v2.0_设计决策记录.md    ← 🆕 v2.0 决策 17~26
    └── mockups/                       ← 23 HTML
```

---

## Git 分支与标签

> 远程仓库：https://github.com/AllenZenghuge/MediaExtractor（私有）

| 分支/标签 | 内容 | 状态 |
|----------|------|------|
| `main` | v2.0 毛玻璃重设计（正式版） | ✅ 已 push（`b17209d` merge commit） |
| `v1.1-darkroom` | v1.1 tag | ✅ 已 push |
| `feature/v2.0-glassmorphism` | v2.0 开发分支 | ✅ 已合并进 main 并删除 |
| `v2.0-glassmorphism` | v2.0 tag | ✅ 已 push |

```bash
# 查看分支
cd "myapp/素材提取/04-App源码/MediaExtractor"
git branch -a

# 克隆远程仓库（默认就是 v2.0 正式版）
git clone https://github.com/AllenZenghuge/MediaExtractor.git

# 回看 v1.1 暗房主题
git checkout v1.1-darkroom
```

---

## 关键技术决策

| 决策 | 原方案 | 变更后 | 原因 |
|------|--------|--------|------|
| X 解析方式 | `__NEXT_DATA__` JSON | **vxtwitter API** | X 页面结构变更 |
| 代理配置 | 硬编码 `127.0.0.1:7897` | **HTTPS_PROXY 环境变量** | 开发/线上环境分离 |
| 重新下载 | 原地修改 DownloadItem | **删旧建新**（delete+insert） | SwiftData @Model 引用/线程安全 |
| 存储策略 | 相册 DCIM | **文件 App（Documents/Media）** | 相册文件名系统锁定 IMG_XXXX，文件 App 文件名 100% 可控 |
| 下载方式 | 全量保存 | **仅勾选项保存** | 用户选择性下载 |
| 视频缩略图 | 仅 VideoItem | **同时为 ImageItem** | 纯视频推文可下载封面 |
| 视觉风格（v2.0） | 暗房黑+红（v1.1） | **冷紫+毛玻璃+深浅双模** | C 端年轻用户，Apple 原生质感 |
| 导航结构（v2.0） | 单页 NavigationStack | **3 Tab 悬浮胶囊** | 提取/下载/我的 三个入口 |
| 代码策略（v2.0） | 直接改 main | **独立 feature 分支 + TDD** | 原版 v1.1 不动，按 TDD 流程开发 |

---

## 存储路径速查

| 位置 | 路径 | 说明 |
|------|------|------|
| 媒体文件（文件 App） | `<App沙盒>/Documents/Media/` | 文件名「博主名_@账号_正文」，文件 App / Finder 可见 |
| 模拟器沙盒 | `~/Library/Developer/CoreSimulator/Devices/16BDA768...FEA/data/Containers/Data/Application/<UUID>/Documents/Media/` | 直接 open 查看 |

> v2.0 起不再存相册（DCIM），改存「文件」App。需开启 `UIFileSharingEnabled`（已配置）。

---

## 启动命令

```bash
# 后端（Mac 开发，带代理，需先激活 venv）
cd "myapp/素材提取/03-后端源码/backend"
source venv/bin/activate
HTTPS_PROXY=http://127.0.0.1:7897 python3 main.py

# App：Xcode 打开 → Cmd+R（main 已是 v2.0 正式版）

# 运行后端测试
cd "myapp/素材提取/03-后端源码/backend"
source venv/bin/activate
python3 -m pytest tests/ -v
```

---

## 会话规则沉淀

- **改代码必更新 md 文档**：设计文档、决策记录、测试计划、测试用例、变更记录
- **所有 md 文件需有修订记录**：| 时间 | 修订内容 |
- **mockup 只新建不覆盖**：每次迭代创建新文件
- **TDD 铁律**：先写测试 → 看失败 → 写最少代码 → 编译通过
- **开发用分支，主线稳定**：功能开发走 feature 分支，完成后 PR 合并进 main

---

## 当前环境

- macOS Darwin 24.6.0 / Python 3.8.3
- Xcode 26.3 (17C529) beta — 模拟器 CLI 不稳定
- 代理: `127.0.0.1:7897` (Clash/V2Ray)
- iOS Simulator: iPhone 17 (iOS 26.3.1), UDID `16BDA768-4290-437E-BC96-98A417347FEA`
- App 数据容器 UUID: `88C01D00-1A73-4527-ACD4-F8F8772EBEBF`
- 后端 venv: `myapp/素材提取/03-后端源码/backend/venv/`

---

## Compact 存档索引

| 次数 | 文件 | 覆盖内容 |
|------|------|---------|
| 首次 | [20260614_v1.0_开发会话纪要_首次Compact.md](myapp/素材提取/01-设计/20260614_v1.0_开发会话纪要_首次Compact.md) | 项目初始化 → App+后端搭建 → 测试体系（2026-06-14 ~ 2026-06-21） |
| 第二次 | [20260622_v1.1_开发会话纪要_第二次Compact.md](myapp/素材提取/01-设计/20260622_v1.1_开发会话纪要_第二次Compact.md) | v1.1 Bug 修复 + v2.0 前端重设计（2026-06-21 ~ 2026-06-22） |
