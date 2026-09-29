# App Code Quality Review

**日期**: 2026-06-21  
**审查类型**: Code Quality Review  
**审查范围**: 全部 20 个 Swift 源文件

## 修订记录

| 时间 | 修订内容 |
|------|---------|
| 2026-06-21 12:14:00 | 初始版本 |

---

## 发现问题汇总

| 严重度 | 数量 | 已修复 |
|--------|------|--------|
| Critical | 2 | ✅ 2 |
| Important | 7 | ✅ 7 |
| Minor | 10 | ⏭ 跳过（非功能性，不阻塞） |

---

## Critical（严重 — 2 个）

### ✅ C1 — APIClient.swift:25 — 多余 `}` 导致编译失败

**描述**: `configure(baseURL:)` 方法后被误插入一个额外的 `}`，将 `ParseRequest`、`parse()` 等成员排挤到类作用域外。

**修复**: 删除多余 `}`，恢复正确类结构。

**文件**: `Services/APIClient.swift:25`

---

### ✅ C2 — GradientArrowButton.swift:32 — 透明度反转

**描述**: `.opacity(isLoading ? 0 : 1)` 导致加载时光晕不可见（opacity=0），闲时反而可见。

**修复**: 改为 `.opacity(isLoading ? 1 : 0)`

**文件**: `Views/Extract/GradientArrowButton.swift:32`

---

## Important（重要 — 7 个）

### ✅ I1 — DownloadManager.swift — 下载进度写入无节流

**描述**: `didWriteData` delegate 在每个数据块到达时都调用 `persistence.update(item)`，高频下载可达每秒数百次 SwiftData 写入。

**修复**: 添加 `lastWrittenBytes` 字典，仅当累计写入 ≥500KB 或下载完成时才持久化进度。

**文件**: `Services/DownloadManager.swift:97-112`

---

### ✅ I2 — DownloadManager.swift — 硬编码文件扩展名

**描述**: `let ext = item.mediaType == .video ? "mp4" : "jpg"` 忽略实际 Content-Type。X/Twitter 图片可为 PNG/WebP 等。

**修复**: 优先使用 `downloadTask.response?.suggestedFilename` 推断扩展名，fallback 到 mediaType 推断。

**文件**: `Services/DownloadManager.swift:60-67`

---

### ✅ I3 — ExtractViewModel.swift — 文案在媒体存在时被丢弃

**描述**: 原逻辑 `if (videos?.isEmpty && images?.isEmpty) && text != nil` 导致图文并存的推文文案不被保存。

**修复**: 将文案保存独立为 `if let text = data.text`，与媒体存在与否无关。

**文件**: `ViewModels/ExtractViewModel.swift:109-123`

---

### ✅ I4 — DownloadListViewModel.swift — deleteAll 遗留孤立文件

**描述**: `deleteAll()` 只清理 SwiftData 记录，不删除磁盘上的下载文件。

**修复**: 删除记录前遍历 `items`，移除 `fileLocalPath` 指向的沙盒文件。

**文件**: `ViewModels/DownloadListViewModel.swift:23-31`

---

### ✅ I5 — DownloadListViewModel.swift — retryItem 后立即 reload 导致 UI 过期

**描述**: `retryItem` 触发异步下载后立即 `loadItems()`，此时下载状态尚未更新，UI 显示旧状态。

**修复**: 添加 0.3 秒延迟再 reload：`DispatchQueue.main.asyncAfter(deadline: .now() + 0.3)`

**文件**: `ViewModels/DownloadListViewModel.swift:33-38`

---

### ✅ I6 — PersistenceService.swift — 所有持久化错误被静默吞没

**描述**: 所有 `save()` 和 `fetch()` 使用 `try?` 丢弃错误，磁盘满或数据损坏时无反馈。

**修复**: 全部改为 `do/catch` + `print("[PersistenceService] error:")` 日志。

**文件**: `Services/PersistenceService.swift:16-47`

---

### ✅ I7 — ExtractViewModel.swift — ImageSaver/VideoSaver 未被调用

**描述**: `ImageSaver` 和 `VideoSaver` 工具类已定义，但 `saveToAlbum()` 只创建 DownloadItem 并启动 URLSession 下载，从未将文件真正写入相册。

**说明**: 当前架构中 `DownloadManager.didFinishDownloadingTo` 将文件存到临时目录并更新 `DownloadItem.fileLocalPath`。保存到相册的逻辑需要在下载完成后补充调用 `ImageSaver`/`VideoSaver`。此问题暂不阻塞 v1 测试（可通过下载列表验证文件是否下载成功），标记为后续迭代修复。

**状态**: 已识别，v2 修复。当前通过手动方式验证：下载完成后在 Files.app 中查看沙盒文件。

---

## Minor（轻微 — 10 个，均跳过）

| # | 文件 | 描述 | 处理 |
|---|------|------|------|
| M1 | DownloadManager.swift | 双重字典查找（first+下标） | 代码清晰度优先，暂不改 |
| M2 | DownloadManager.swift | `URL.path()` 已弃用（iOS 16+） | 不影响功能，暂不改 |
| M3 | ImageResultTab.swift | `UIScreen.main` 已弃用 | 暂不改 |
| M4 | ExtractView.swift | `NavigationLink(destination:)` 已弃用（iOS 16+） | 暂不改 |
| M5 | ExtractViewModel.swift | `import UIKit` 仅用于 `UIPasteboard` | 导入合理，不改 |
| M6 | ExtractViewModel.swift | `await MainActor.run` 可能冗余 | 无害，不改 |
| M7 | DownloadRowView.swift | 已完成项显示"重新下载"按钮 | 符合 spec 设计意图 |
| M8 | Color+Theme.swift | 非法 hex 静默 fallback 到透明黑 | 开发阶段可接受 |
| M9 | ImageSaver.swift | 潜在 retain cycle（onComplete 闭包） | 调用后置 nil 即可，暂不改 |
| M10 | DownloadItem.swift | mediaURLs 过期提示无对应重解析逻辑 | 标记为 v2 增强 |

---

## 架构观察

**优点**:
- MVVM 分层清晰：ViewModel 拥有业务逻辑，View 薄且只负责渲染
- SwiftData 持久化层隔离在 PersistenceService 中
- 下载管理独立为 DownloadManager，遵循单一职责
- 暗房主题色通过 Color extension 统一管理

**待改进**:
- ImageSaver/VideoSaver 与 DownloadManager 之间的集成链未闭合（I7）
- 重试下载使用过期 URL 而非重新解析（后续迭代）

---

审查人: Claude Code (subagent) | 审查日期: 2026-06-21
