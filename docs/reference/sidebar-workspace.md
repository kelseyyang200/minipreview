# 侧边栏工作目录模块（多根工作区）

## 功能概述

左侧边栏（264px，可收起）的第一个标签页。像 VSCode 多根工作区一样：可同时添加**多个**项目目录，每个目录一个可折叠分组；往任何目录里丢视频，即可在侧边栏直接点开播放。

## UI 结构

```
aside#sidebar
├── .sidebar-tabs（两个标签：工作目录 / 最近打开）
└── #sidebarPanel-workspace
    ├── #fsUnsupportedHint      不支持 FS API 时的提示（Firefox/Safari）
    ├── #pickDirBtn             📂 添加工作目录（可反复点，追加目录）
    ├── #workspaceToolbar       筛选框（跨全部目录）+ 🔄全量重扫
    └── #workspaceFileList      目录分组列表
        └── .ws-dir-group       每目录一组（.open 展开 / 默认按折叠状态）
            ├── .ws-dir-header  ▸/▾ 📁 目录名（N）+ [🔄重扫][▶全部加载][×移除]
            └── .ws-dir-files   .ws-file 文件项 或 待授权/空提示
```

- 待授权目录：标题显示 🔓，actions 只有 [🔐 授权][× 移除]。
- 筛选时：各组只显示匹配项，无匹配的组隐藏，组强制展开。

## 核心函数

| 函数 | 职责 |
| --- | --- |
| `pickWorkspaceDir()` | `showDirectoryPicker` 选目录 → `isSameEntry` 对已存在目录去重（重复弹提示）→ 写一条 `dir` 记录 → 追加到 workspaceDirs → 扫描 |
| `restoreWorkspaceDirs()` | 启动恢复目录列表；**兼容旧版**：`mp:workspaceDirId`（单目录 key）自动迁移为 `mp:workspaceDirs` 数组 |
| `regrantWorkspaceDir(dirId)` | 单目录重新授权（用户手势内 requestPermission） |
| `removeWorkspaceDir(dirId)` | 从工作区移除；**IndexedDB 的 dir 记录保留**，最近打开的该项目记录仍可重开 |
| `toggleDirCollapsed(dirId)` | 折叠状态存 localStorage `mp:wsDirCollapsed`（id 数组） |
| `scanOneDir(dir)` / `scanWorkspace()` | 单目录扫描 / 全部重扫；扫描时给每个文件 entry 注入 `dirId`/`dirName` |
| `rescanOneDir(dirId)` | 分组内的 🔄 按钮 |
| `loadWorkspaceEntry(entry)` | 单文件加载，项目记录用 `entry.dirId/dirName` |
| `loadAllWorkspace(dirId)` | **按目录**顺序加载全部（preserveOrder 与 relPaths 顺序严格一致） |
| `updateWorkspaceUI()` | 两态：无目录引导 / 有目录显示筛选框；分组渲染交给 renderWorkspaceList |

## 状态与存储

```js
let workspaceDirs = [];  // [{id, name, handle, perm, files, collapsed}]
```

| 介质 | 键 | 内容 |
| --- | --- | --- |
| localStorage | `mp:workspaceDirs` | 目录记录 id 数组（有序） |
| localStorage | `mp:wsDirCollapsed` | 折叠的目录 id 数组 |
| localStorage | ~~`mp:workspaceDirId`~~ | 旧版单目录 key（自动迁移后删除） |
| IndexedDB | `dir:` 记录 | 每个添加过的目录一条（含句柄），供工作区与最近打开共用 |

## 授权生命周期（重点）

- 目录授权浏览器重启后失效（`prompt`），每个目录独立授权一次（分组内 🔐 按钮）。
- 移除工作区目录不删 db 记录；重新「添加工作目录」选同一个文件夹会新建 dir 记录（isSameEntry 只对当前列表去重）。
- 最近打开的 dirfile/dirall 记录带 dirId，重开走 `openRecentFromDir` → `dbGet(dirId)` → 授权 → 按路径取文件，与多目录完全兼容。

## 浏览器兼容

- `showDirectoryPicker` 不存在（Firefox/Safari/旧 Chromium）→ 显示不支持提示，最近打开的 URL 类记录仍可用。

## 常见改动点

- **调扫描深度/上限**：`scanDirRecursive` 的 `depth > 4` / `out.length >= 800`。
- **加视频格式**：模块顶部 `VIDEO_EXTS`。
- **跨目录"加载全部"**：目前按目录分组加载；如需全局连播，把所有已授权目录的 files 按 workspaceDirs 顺序拼接成一个 dirall 记录即可。
