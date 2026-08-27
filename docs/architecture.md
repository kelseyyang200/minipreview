# 技术方案与架构

## 总体形态

- 零依赖纯静态单页，所有 CSS / JS 内联在 `index.html`。
- 全局变量共享状态（无模块化），函数声明提升，跨区域调用靠全局函数名。
- 目标浏览器：Chrome / Edge 86+（工作目录依赖 File System Access API）；其余浏览器降级可用（详见各模块文档）。

## 页面布局

```
┌────────────────────────── toolbar（60px）──────────────────────────┐
│ logo │ URL输入+加载+本地文件 │ 状态提示 │ 导出批注 ▼ │ 分享批注     │
├──────┬───────────────────────────────────────┬────────────────────┤
│side- │ video-section                          │ annotation-panel   │
│bar   │  video-wrapper（video + uploadHint）   │  panel-header      │
│264px │  timeline-section（进度/标记/预览头）   │  manualAddSection  │
│      │  controls（播放/倍速/逐帧/加标签）      │  annotations-list  │
│      │                                        │ 360px              │
└──────┴───────────────────────────────────────┴────────────────────┘
右上角 fixed：version-badge（版本徽章/更新日志/一键刷新）
左中 fixed：sidebar-toggle（收起/展开，body.sidebar-collapsed 控制）
```

## 核心全局状态（script 顶部）

| 变量 | 含义 |
| --- | --- |
| `video` | 主 `<video>` 元素 |
| `videos` / `currentVideoIndex` / `totalDuration` | 多视频播放列表（连续播放 + 分段时间轴） |
| `annotations` | 当前项目批注数组；多视频模式下 time 是全局时间，localTime/videoIndex 是视频内定位 |
| `currentProject` | 当前打开的项目记录（含 id），批注自动保存挂在这个 id 上 |
| `workspaceDirHandle` / `workspaceFiles` / `workspacePerm` | 工作目录句柄、扫描结果、授权状态 |
| `sideDb` | IndexedDB 连接（库名 `minipreview-db`，store `projects`，keyPath `id`） |
| `APP_VERSION` / `CHANGELOG` | 版本号与更新日志（部署前必须更新，见 agent.md 规则 1） |

## 关键数据流

### 项目加载（所有入口统一走 loadVideoFiles）

```
入口（4 类）                      → 统一处理
─────────────────────────────────────────────────────
<input> 选择（loadLocalFiles）   → loadVideoFiles(files)
showOpenFilePicker（pickLocalFiles） → loadVideoFiles(files, {handles})
工作目录单文件（loadWorkspaceEntry） → loadVideoFiles([file], {project: dirfile记录})
最近打开重开（openRecentProject）  → loadVideoFiles(files, {project / handles})
拖拽（handleDropUpload→handleVideoUpload）→ 单视频模式（不走 videos[] 播放列表）
URL / 微盘（loadVideoFromUrl / enableWeDriveMode）→ openProjectRecord 直接记录
```

`loadVideoFiles` 内部顺序：排序 → `openProjectRecord`（写最近记录 + 恢复批注）→ 逐个 `URL.createObjectURL` 探测时长构建 `videos[]` → `loadVideoByIndex(0)`。

### 批注自动保存（防抖 500ms）

任何批注变化 → `renderAnnotations()`（渲染统一出口）→ 末尾 `scheduleAnnotationSave()` → 防抖写 IndexedDB（记录的 `annotations` 字段）。

### 存储设计

| 介质 | 键 | 内容 |
| --- | --- | --- |
| IndexedDB `projects` | 记录 id | 项目记录（类型/名称/句柄/批注/lastOpened/annotationCount） |
| localStorage | `mp:workspaceDirId` | 当前工作目录对应的记录 id |
| localStorage | `mp:sidebarCollapsed` / `mp:sidebarTab` | 侧边栏 UI 偏好 |
| sessionStorage | `mp:forceRefreshed` | 版本守卫会话标记（防无限刷新） |

项目记录 id 规则（确定性 id，重开同名文件自动匹配旧批注）：

| 类型 | id | 重开方式 |
| --- | --- | --- |
| files | `files:` + 排序后文件名 join `\|` | 存 fileHandles 直接重开；无句柄则重新选文件 |
| dirfile | `dirfile:` + dirId + `\|` + 相对路径 | 从 dir 记录的目录句柄按路径取文件 |
| dirall | `dirall:` + dirId + `\|` + 相对路径列表 | 同上，逐个取 |
| url | `url:` + 完整 URL | 直接加载 |
| wedrive | `wedrive:` + 完整 URL | 重新进微盘模式 |
| dir | `dir:` + 目录名 + `:` + 时间戳 | 工作目录授权记录（不出现在最近列表） |

## 模块索引（详解见 reference/）

- [reference/sidebar-workspace.md](reference/sidebar-workspace.md) — 工作目录
- [reference/recent-projects.md](reference/recent-projects.md) — 最近打开 + 批注缓存
- [reference/version-refresh.md](reference/version-refresh.md) — 版本检测 + 强刷
- [reference/upstream-bugs.md](reference/upstream-bugs.md) — 上游 bug 修复清单
