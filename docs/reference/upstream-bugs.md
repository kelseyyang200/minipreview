# 对上游的改动清单与 PR 策略

上游：[kelseyyang200/minipreview](https://github.com/kelseyyang200/minipreview)（基线 commit `45d52f0`，2026-08-26 克隆）。

## 我方改动全量清单（v1.0.0 → v1.5.0）

### A. 上游 Bug 修复（上游存在的问题）

| # | 位置 | 上游行为 | 我方修复 |
| --- | --- | --- | --- |
| 1 | `deleteCurrentEpisode()` | 删除某集后调 `loadVideo(currentVideoIndex)`（把索引当 URL）且调用不存在的 `updateEpisodeList()` → ReferenceError，删除流程崩溃 | 改 `loadVideoByIndex()`；移除不存在函数的调用 |
| 2 | `exportCurrentEpisodeImage()` | `annotations[v.name]` 把数组当字典用，永远取不到 → 有批注也提示"没有批注" | `annotations.filter(a => a.videoName === v.name)` |
| 3 | `handleKeyboard()` | 焦点在输入框（URL 框等）时按 Space/←/→/M 触发播放控制且 preventDefault，无法正常输入 | 事件开头对 INPUT/TEXTAREA 目标直接 return |
| 4 | `loadVideo(url)` | 从微盘模式切回普通视频后，微盘手动批注区残留、`currentWeDriveUrl` 未清空导致跳转批注仍走微盘分支 | loadVideo 时隐藏清空该区域、置空标记 |
| 5 | `loadedmetadata` | 加载完成不重画批注标记（标记位置依赖 duration） | loadedmetadata 里补 `renderAnnotations()` |

### B. 新功能（适合贡献给上游）

| # | 功能 | 版本 | 说明 |
| --- | --- | --- | --- |
| 1 | 多根工作区侧边栏 | 1.1.0/1.2.0 | 左侧边栏 + 多目录（类 VSCode）：添加/移除/折叠/筛选，子目录递归扫描，按目录顺序加载全部；依赖 File System Access API（Chromium 86+，其他浏览器降级提示） |
| 2 | 最近打开 + 批注本地缓存 | 1.1.0 | IndexedDB 持久化项目记录（含文件句柄）与批注，重开自动恢复；确定性项目 id（同名文件自动匹配旧批注） |
| 3 | 会话自动恢复 | 1.3.0 | 刷新/重开自动回到上次项目+批注+播放位置；权限失效时提示条一键恢复 |
| 4 | 普通导入/导出 | 1.4.0 | 批注 JSON 小文件导出/导入（跨设备迁移批注） |
| 5 | 超级导入/导出 | 1.4.0 | 批注+原视频打包 ZIP 完整包（零依赖 store 模式 ZIP 引擎，流式 CRC32）；链接类项目携带来源链接 |
| 6 | 左右键步长滑动条 | 1.4.1 | 1帧~10秒 7 档可调（默认 1 秒），偏好记忆，方向键与 ⏪⏩ 共用 |
| 7 | 黑屏检测提示 | 1.4.2 | `videoWidth===0` 检出 H.265 等解不出的编码，给出转码/换 Edge 建议 |
| 8 | 版本检测 + 更新日志 + 一键刷新 | 1.1.0/1.3.1 | 远端版本核对自动强刷（四道防线防循环）+ 更新日志徽章 + 工具栏一级刷新按钮；适配 GitHub Pages 等静态托管的缓存问题 |

### C. 自用功能（不进 PR）

| # | 功能 | 原因 |
| --- | --- | --- |
| 1 | 一键转码（本机 Python+ffmpeg 助手 + 页面按钮 + 自助下载包） | 依赖用户自装本机助手服务，超出上游"零依赖轻量"定位 |

## PR 策略（已执行 ✅）

**PR 地址：https://github.com/kelseyyang200/minipreview/pull/1**（fork：lyzbcy/minipreview，分支 feat/memory-and-workspace，2026-08-27 提交）

**单个综合 PR**：包含 A 全部 + B 全部。从分支 `feat/memory-and-workspace` 提交到上游 `main`。

- 剔除 C（转码助手相关代码整体移除；黑屏检测保留，文案改为通用建议）
- CHANGELOG/版本机制保留并在 PR 描述说明（上游可自行取舍）
- PR 描述用中文，分节列 bug 修复 / 新功能 / 浏览器兼容边界 / 测试情况

## 后续同步注意

- 上游若有更新：以 `git fetch origin` 对比，我方改动集中在 index.html 单文件，冲突时按本清单逐块核对。
- 线上版（help.wshoto）始终是全量版（含转码）；PR 分支只是贡献用的精简快照。
