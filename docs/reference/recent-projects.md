# 最近打开与批注本地缓存模块

## 功能概述

1. 「最近打开」标签页：自动记录所有加载过的项目（目录视频 / 本地文件 / 直链 / 微盘 / 目录全部），点击重开，单条删除或清空。
2. 批注按项目自动保存到 IndexedDB，重开同一项目自动恢复——解决上游"刷新丢批注"的核心痛点。
3. **会话恢复**：刷新 / 重开页面自动回到上次的现场（项目 + 批注 + 播放位置），无需手动点最近记录。

## 会话恢复机制（1.3.0+）

| 环节 | 实现 |
| --- | --- |
| 记住上次项目 | `openProjectRecord` 写 `localStorage['mp:lastProjectId']` |
| 自动恢复入口 | `initSidebarApp` 末尾 `restoreLastSession()`；分享链接模式（`?data=`）不抢现场 |
| 免打扰判断 | `canAutoRestore`：url/wedrive 直接恢复；本地文件类逐句柄 `queryPermission`，**全部 granted 才自动恢复**（避免刷新时弹权限框） |
| 权限失效时 | `showRestoreBar`：视频区顶部提示条「检测到上次的项目「XX」」，点击（手势）→ `openRecentProject` 内 requestPermission |
| 播放位置保存 | `schedulePosSave`（timeupdate 防抖 2s）+ `savePosNow`（beforeunload / visibilitychange 强制），存 `localStorage['mp:pos:<projectId>']` = `{videoIndex, time}` |
| 播放位置恢复 | `pendingSeekRestore` 标志：仅自动恢复和提示条恢复时 seek 回上次位置；用户日常点开从头播放 |
| 批注防丢 | `flushAnnotationSave`：beforeunload / visibilitychange hidden 时把防抖中的批注立即落库 |

## 项目记录结构（IndexedDB `projects` store）

```js
{
  id: string,            // 确定性 id，规则见 architecture.md
  type: 'files'|'dirfile'|'dirall'|'url'|'wedrive'|'dir',
  name: string,          // 展示名
  videoCount: number,
  annotationCount: number,
  annotations: [...],    // 批注数组（自动保存写入）
  lastOpened: timestamp,
  // 以下按类型可选：
  fileHandles: [...],    // files：showOpenFilePicker / 拖拽拿到的句柄
  dirId / dirName,       // dirfile/dirall：来源目录
  relPath / relPaths,    // dirfile/dirall：相对路径（可含子目录）
  url,                   // url/wedrive
  handle,                // dir 记录：目录句柄
}
```

## 核心函数

| 函数 | 职责 |
| --- | --- |
| `openProjectRecord(rec)` | **记录/合并 + 恢复批注的唯一入口**。合并时保护已有 fileHandles 不被空数组覆盖；设置 `currentProject` 与 `mp:lastProjectId`；从 existing.annotations 恢复批注 |
| `buildFilesProject(files, handles)` | files 类型确定性 id：排序后文件名 join `\|`（重选同名文件 → 同 id → 旧批注自动恢复） |
| `scheduleAnnotationSave()` / `saveAnnotationsNow()` / `flushAnnotationSave()` | 批注防抖 500ms 保存 / 立即保存 / 页面关闭时强制落库 |
| `schedulePosSave()` / `savePosNow()` / `restorePlaybackPos()` | 播放位置防抖保存 / 强制保存 / seek 回上次位置（依赖 pendingSeekRestore 标志） |
| `restoreLastSession()` / `canAutoRestore(rec)` / `showRestoreBar(rec)` | 会话自动恢复 / 免打扰判断（全句柄 granted 才静默恢复）/ 权限失效提示条 |
| `openRecentProject(id)` | 最近列表点击入口，按 type 分发 |
| `openRecentFromDir(rec)` | 从 dir 记录句柄按 `getFileByPath`（逐级 getDirectoryHandle）取文件 |
| `openRecentFiles(rec)` | 有句柄直接重开（逐个 ensureReadPermission）；无句柄重开文件选择器（同名匹配恢复批注） |
| `renderRecentList()` | 最近 30 条按 lastOpened 倒序；⊘ 标记不可直接重开的记录 |

## 关键设计

- **批注恢复时机**：`openProjectRecord` 在视频加载**前**执行（annotations 先就位），视频加载完 `renderAnnotations()` 再画时间轴标记（标记位置依赖 totalDuration/duration）。
- **多视频顺序稳定性**：dirall 记录的 relPaths 顺序 = 首次加载顺序；重开时 `preserveOrder: true`，批注里的 videoIndex 才能对上。
- **句柄降级存储**：`dbPut` 捕获结构化克隆失败（个别环境句柄不可存），去句柄重存——记录仍可见，重开时引导重新选文件（⊘）。
- **分享链接优先**：URL 带 `?data=`（分享批注）时由 `loadAnnotationsFromUrl` 主导，不与项目缓存互相覆盖；用户主动加载新项目才会切换。

## 已知边界

- `addMoreVideos`（追加视频）不更新项目记录的 videoCount，追加视频的批注仍会保存（id 不变），但记录展示的数量是首次加载时的。
- 记录条数无清理策略（渲染时截取前 30），本地工具可接受。
