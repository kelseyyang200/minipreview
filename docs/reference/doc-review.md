# 文档审阅模块（Markdown 剧本，1.6.0+）

## 功能概述

审 Markdown / txt 剧本：按行渲染（行号稳定，批注锚定**行号**），复用整个批注生态（面板、IndexedDB 缓存、最近打开、会话恢复、超级导入导出、AI Prompt 导出）。

## 模式切换

`body.doc-mode` 类控制一切：隐藏 video/uploadHint/timeline/controls，显示 `#docViewer` 和 `.doc-toolbar`。
判定统一用 `isDocMode()`（body class），**不要**用 `currentProject.type === 'doc'`（工作目录来的记录是 `dirfile` + `isDoc: true`）。

## 入口分流（全部走 loadVideoFiles 开头 / 独立函数）

| 入口 | 路径 |
| --- | --- |
| 文件选择 / 拖拽 / 最近打开 files / 超级导入 | `loadVideoFiles` 开头 `isDocFilename(files[0].name)` → `loadDocumentFile(file, handle, opts.project)` |
| 工作目录条目（📄 图标） | `loadWorkspaceEntry` doc 分支 → `loadDocumentFile(file, null, dirfile记录)` |
| URL（.md/.txt 结尾） | `loadVideoFromUrl` → `loadDocumentFromUrl` → fetch 文本 → `loadDocumentFile`（记录带 `docUrl`） |

## 项目记录

- id：`doc:<文件名>`；type `doc`；本地来源带 `fileHandles`，URL 来源带 `docUrl`
- 工作目录来源：type `dirfile` + `isDoc: true` + `dirId/relPath`（重开走目录句柄）
- 批注结构：`{ id, lineNumber, content }`（无 time 字段）

## 批注交互

- 单击行选中（`selectedDocLine`），**双击行**直接加批注，`M` 给选中行加批注
- `renderDocument()`：行渲染 + 事件委托；`renderAnnotations()` 末尾 doc 模式会调它刷新行高亮（无递归：renderDocument 不回调 renderAnnotations）
- `jumpToAnnotation` doc 分支 → `jumpToDocLine`（scrollIntoView + flash 动画）
- 批注面板显示「第 N 行」

## 渲染细节

- 保留行结构（每行一个 div，行号锚定），行内轻量 Markdown：`` `code` ``、`**粗**`、`*斜*`；块级按前缀加类：`#`→h1/h2/h3、`>`→quote、`---`→divider（语法符号本身保留显示，保真优先）
- `escapeHtml` 先行，再做行内替换（防注入）

## AI Prompt 导出（`exportAiPrompt`，菜单「🤖 AI 修改指令」）

- **文档模式**：组装「修改要求（行号批注列表）+ 剧本原文（带行号）+ 输出要求（完整输出/保持格式/修改标记）」的完整 prompt，下载 .md 并复制剪贴板——直接丢给 AI 改剧本
- **视频模式**：批注整理成「给剪辑助理的修改清单」prompt
- 视频模式的文本导出（`exportAsText`）兼容 doc：按行号输出

## 已知边界

- 超大文档（万行级）一次性 innerHTML 渲染，可能卡顿；无虚拟滚动
- URL 文档受跨域限制；docUrl 记录后重开走 fetch（内容变了批注行号可能漂移，属预期）
- 多选文档只加载第一个
