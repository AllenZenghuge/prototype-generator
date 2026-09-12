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
| 2026-09-12 22:30:00 | 补记 08-24~09-12 三平台解析落地：X 直连 vxtwitter / 小红书抓 HTML / 抖音 WKWebView 借 JS 签名；App 端完全脱离 Python 后端；新增待验证事项与遗留问题 |
| 2026-09-12 23:13:00 | 抖音真机验证通过：定位根因「移动 UA 被带到 iesdouyin 分享页，该页不请求 aweme/detail」，修复为伪装桌面 UA；错误提示加诊断串并支持长按复制 |
| 2026-09-12 23:24:00 | 抖音图文作品（note）真机验证通过：定位 `/note/` 路径不请求 detail，修复为导航拦截改写 `/video/{id}`；V-03 完成；修正 `_ROUTER_DATA` 描述 |

---

## 一句话总结

iPhone App（SwiftUI + SwiftData），实现 **X / 小红书 / 抖音** 三平台无水印视频/图片/文案提取。**App 端已完全本地直连解析，不再调用 Python 后端；v2.0 毛玻璃重设计为正式版，v1.1 暗房主题由 `v1.1-darkroom` tag 保留。**

---

## 当前状态总览

| 模块 | 版本 | 分支 | 状态 |
|------|------|------|------|
| **v1.1 暗房主题** | v1.1 | tag `v1.1-darkroom` | ✅ 历史版本，保留可回看 |
| **v2.0 毛玻璃重设计** | v2.0 | `main` | ✅ 已合并，正式版 |
| **三平台解析** | v2.1 | `main` | ✅ 三平台真机验证通过 |
| **Python 后端** | — | `main` | ⚠️ 源码保留在 `03-后端源码/`，但 App 端已不再调用 |
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

## 三平台解析（v2.1，✅ 三平台均已真机验证）

时间跨度：2026-08-24 ~ 2026-09-12
架构结果：**App 端完全本地直连解析，不再调用 Python 后端**（后端源码保留在 `03-后端源码/`，当前不参与运行）。

| 平台 | 方案 | 关键点 |
|------|------|--------|
| **X** | 直连 `api.vxtwitter.com`（`VxTwitterParser`） | 下载直连 twimg.com；翻墙由手机 VPN 负责，后端不提供 |
| **小红书** | 抓 HTML + 正则提取 `masterUrl`（`XhsParser`） | 无需签名；封面优先取 `imageList` 的 `urlDefault`（`og:image` 是平台默认图） |
| **抖音** | 隐藏 WKWebView（**伪装桌面 UA**）+ 注入 JS hook 拦截 detail 响应（`DouyinWebParser`） | 借抖音自己的 JS 生成 `a_bogus` 签名；**必须伪装桌面 UA**，否则进不去会请求 detail 的 PC 版页面 |

### 抖音方案：为什么用 WKWebView

核心判断：`a_bogus` 签名由**抖音自己的 JS 生成**，自建算法约 300 行且每两周失效一次；借 JS 签名可随平台自动适配。

流程：WKWebView 加载链接 → 抖音 JS 生成签名请求 detail → 注入 JS hook `XHR`/`fetch` 捕获响应 → `postMessage` 回 Swift → 取 `aweme_detail.video.play_addr.url_list[0]`（无水印）。首次解析 2-5 秒，25 秒超时兜底。

### ⚠️ 关键坑：必须伪装桌面 UA（2026-09-12 定位并修复）

**现象**：真机上抖音解析失败，提示「25 秒内未捕获到 aweme/detail 响应」。

**根因**：移动 UA 下，**无论短链 `v.douyin.com/xxx` 还是完整链接 `www.douyin.com/video/{id}`**，WKWebView 最终都会落到 `www.iesdouyin.com/share/video/{id}` —— 移动端 reflow 分享页。该页是「打开抖音 App」引导页（`isAutoOpenApp: true`、`page_name: reflow_video`），其 JS 里 **0 处 `aweme/detail`**（只有 `aweme/v1/playwm`），hook 永远等不到响应，25 秒超时后返回 nil。

**证据**：把分享页 7 个 JS chunk 全量下载并 grep，`aweme/detail` 出现 0 次；而桌面浏览器加载同一链接会进入 `www.douyin.com/video/{id}`，那里才请求 `aweme/v1/web/aweme/detail/`（实测 200）。

**修复**：`webView.customUserAgent` 设为桌面 Chrome UA，让抖音返回 PC 版页面（见 `DouyinWebParser.desktopUserAgent`）。

**诊断手段**：`DouyinWebParser.parse` 失败时返回诊断串（区分「超时 / 非 JSON / 无 aweme_detail 字段 / 取不到地址」四种环节，并带上 WebView 最终停留的 URL），由 `APIClient` 拼进错误提示，界面支持长按复制。**下次抖音再挂，先看这句里的 finalURL 判断页面落点。**

### ⚠️ 关键坑 ②：图文作品必须改写 `/note/` → `/video/`（2026-09-12 修复）

**现象**：视频链接修好后，图文作品（图集）仍报「25 秒内未捕获到 aweme/detail 响应；WebView 最终停留于 `https://www.douyin.com/note/7683370298066856305`」。

**根因**：图文作品走 `/note/{id}` 路径，**该路径在 PC 端不请求 `aweme/detail`，页面也不渲染内容**；同一个作品 ID 换成 `/video/{id}` 就会正常调用详情接口。

**证据**：同一作品 ID 分别加载 `/note/{id}` 与 `/video/{id}` 抓网络 —— 前者的 aweme 接口里**没有 `detail`**（只有 `aweme/post`、`aweme/related`），后者有 `aweme/v1/web/aweme/detail/`。

**修复**：`DouyinWebParser` 实现 `WKNavigationDelegate`，导航过程中 URL 只要含 `/note/` 就取消并改写为 `https://www.douyin.com/video/{id}` 重新加载。对短链和完整链接都透明。

### 抖音已排除方案（勿重复尝试）

| 方案 | 失败原因 |
|------|---------|
| PC UA 抓 HTML | 空壳反爬（只有 2 个空 script） |
| iPhone UA 抓 SSR 分享页 | 只有元数据，无视频地址 |
| 移动端 `_ROUTER_DATA` | 字段其实**存在**，但只有 webId/itemId 等壳数据，无视频地址 |
| aweme.snssdk.com 直链 | 用数字 ID 不返回直链 |
| Chrome cookie（`s_v_web_id`）+ detail API | 返回空（**需要签名**，不只是 cookie） |
| 第三方 API（小渡/lp8） | 需注册 Token/ckey |
| 移动 UA 加载（短链、完整链接都一样） | 被带到 `iesdouyin.com/share/video/` 移动分享页，该页不请求 `aweme/detail` |
| 图文作品走 `/note/{id}` 路径 | PC 端既不请求 `aweme/detail`、也不渲染内容，须改写为 `/video/{id}` |

### 本节变动文件

| 类别 | 文件 | 内容 |
|------|------|------|
| 🆕 | `Services/MultiPlatformParser.swift` | `PlatformDetector` 平台识别 + `XhsParser` 小红书解析 + 正则辅助 |
| 🆕 | `Services/VxTwitterParser.swift` | `VxTwitterResponse` 模型 + `TweetURLParser` + `VxTwitterConverter` |
| 🆕 | `Services/DouyinWebParser.swift` | WKWebView 拦截抖音 detail 响应 |
| 🆕 | `Utilities/URLParser.swift` | 从「文字+链接」分享文案中正则提取 URL |
| ✏️ | `Services/APIClient.swift` | 由「调后端」改为「按平台本地路由」 |
| ✏️ | `Info.plist` | 手动 Info.plist 修复 ATS（`NSAllowsArbitraryLoads`）+ 文件共享 |

### 待验证事项

| 编号 | 事项 | 说明 |
|------|------|------|
| ~~V-01~~ | ~~真机验证抖音~~ | ✅ 2026-09-12 真机通过（修复：伪装桌面 UA） |
| V-02 | 小红书边界 | 纯图笔记、多图、需登录的笔记 |
| ~~V-03~~ | ~~抖音图集~~ | ✅ 2026-09-12 真机通过（修复：`/note/` 改写为 `/video/`） |

### 遗留问题

- `MultiPlatformParser.swift:156` 的 `enum DouyinParser`（早期 HTML + `playwm→play` 去水印方案）已无任何调用点，是被 `DouyinWebParser` 取代后的死代码（约 45 行），待清理。

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
│       │   ├── Extract/               ← v2.0 冷紫+毛玻璃（9 文件）
│       │   ├── DownloadList/          ← v2.0 毛玻璃卡片（3 文件）
│       │   ├── Profile/               ← 🆕 C3 v4「我的」
│       │   └── Settings/              ← 🆕 设置+登出+注销
│       ├── Services/        ← 6 文件（APIClient 多平台路由、MultiPlatformParser、DouyinWebParser、VxTwitterParser、DownloadManager、PersistenceService）
│       └── Utilities/       ← 6 文件（Color+Theme 双色板 + URLParser/BatchDownloadBuilder/FilenameSanitizer/MediaStorage/PasteboardHelper）
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
| `main` | v2.0 毛玻璃 + 三平台解析（正式版） | ✅ 已 push（最新 `1d7bf7b`） |
| `v1.1-darkroom` | v1.1 tag | ✅ 已 push |
| `feature/v2.0-glassmorphism` | v2.0 开发分支 | ✅ 已合并进 main 并删除 |
| `v2.0-glassmorphism` | v2.0 tag | ✅ 已 push |

> ⚠️ 偏差记录：08-24 ~ 09-12 的三平台改动（`5b60891` ~ `1d7bf7b`，共 8 个 commit）**直接提交在 `main` 上**，未走 feature 分支，与下方会话规则「开发用分支，主线稳定」不符。后续开发建议恢复分支流程。

```bash
# 查看分支
cd "myapp/素材提取/04-App源码/MediaExtractor"
git branch -a

# 克隆远程仓库（默认即最新正式版：v2.0 + 三平台）
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
| 后端角色（v2.1） | App 调 Python 后端 | **App 端完全本地直连** | 龙哥明确「翻墙是用户手机 VPN 的事，后端不提供翻墙」；三平台解析均可客户端完成 |
| 抖音解析（v2.1） | 抓 HTML / 自建 `a_bogus` 签名 | **WKWebView 借抖音 JS 签名** | 自建约 300 行且每两周失效，借 JS 签名自动适配 |
| 上架架构（远期） | — | **X 直连 + 小红书/抖音走后端** | 国内服务器不翻墙，规避 App 内 WebView 抓取风险 |

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
# App（当前唯一运行入口）：Xcode 打开 → Cmd+R
# 路径：myapp/素材提取/04-App源码/MediaExtractor（main = v2.0 + 三平台正式版）
# X 需手机 VPN；小红书/抖音国内直连，无需 VPN。

# 后端（⚠️ App 已不再调用，仅历史参考 / 跑后端测试用）
cd "myapp/素材提取/03-后端源码/backend"
source venv/bin/activate
HTTPS_PROXY=http://127.0.0.1:7897 python3 main.py

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
- Xcode 26.3 (17C529) beta — 模拟器 CLI 不稳定，单测用 Xcode Cmd+U
- 代理: `127.0.0.1:7897` (Clash/V2Ray)
- 网络：**X 需手机 VPN**；小红书/抖音为国内平台，无需 VPN
- iOS Simulator: iPhone 17 (iOS 26.3.1), UDID `16BDA768-4290-437E-BC96-98A417347FEA`
- App 数据容器 UUID: `88C01D00-1A73-4527-ACD4-F8F8772EBEBF`
- 后端 venv: `myapp/素材提取/03-后端源码/backend/venv/`
- git 推送：有代理时走 `127.0.0.1:7897`，代理未开时直连 push：`git -c http.proxy= -c https.proxy= push`

---

## Compact 存档索引

| 次数 | 文件 | 覆盖内容 |
|------|------|---------|
| 首次 | [20260614_v1.0_开发会话纪要_首次Compact.md](myapp/素材提取/01-设计/20260614_v1.0_开发会话纪要_首次Compact.md) | 项目初始化 → App+后端搭建 → 测试体系（2026-06-14 ~ 2026-06-21） |
| 第二次 | [20260622_v1.1_开发会话纪要_第二次Compact.md](myapp/素材提取/01-设计/20260622_v1.1_开发会话纪要_第二次Compact.md) | v1.1 Bug 修复 + v2.0 前端重设计（2026-06-21 ~ 2026-06-22） |
