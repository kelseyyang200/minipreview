# MiniPreview 开发入口（AI 必读）

任何 AI 接手本项目开发前，先读完本文件（约 2 分钟），再按需跳转对应模块文档，不要直接通读 `index.html`（3300+ 行）。

## 项目一句话

纯静态单文件视频审片工具（原生 HTML/CSS/JS，无构建、无依赖），在原作者 [kelseyyang200/minipreview](https://github.com/kelseyyang200/minipreview) 基础上增加了侧边栏工作目录、最近打开、批注本地缓存、版本检测与一键刷新。

- 线上地址（用户自用）：https://help.wshoto.com/resource/6fe1735d177343599a25c1f02a4c6d66/d94dca03.html
- 当前版本：`APP_VERSION`（见 index.html，与 CHANGELOG 第一条一致）
- 全部代码在 `index.html` 一个文件里：`<style>` 区 + `<body>` 结构 + 一个 `<script>`（函数声明为主，全局变量共享状态）

## 目录管理（渐进式披露）

| 要做什么 | 先读 |
| --- | --- |
| 了解整体架构 / 数据流 / 存储设计 | [architecture.md](architecture.md) |
| 改侧边栏、工作目录扫描、目录授权 | [reference/sidebar-workspace.md](reference/sidebar-workspace.md) |
| 改最近打开、项目记录、批注自动保存与恢复 | [reference/recent-projects.md](reference/recent-projects.md) |
| 改版本号、强刷逻辑、更新日志、一键刷新 | [reference/version-refresh.md](reference/version-refresh.md) |
| 改导入/导出（批注 JSON / 超级 ZIP 完整包） | [reference/import-export.md](reference/import-export.md) |
| 改一键转码 / 本机转码助手 | [reference/transcode.md](reference/transcode.md) |
| 改文档审阅（Markdown 剧本 / AI Prompt 导出） | [reference/doc-review.md](reference/doc-review.md) |
| 给上游提 PR / 了解改了上游哪些 bug | [reference/upstream-bugs.md](reference/upstream-bugs.md) |
| 看待办 / 路线图 / 历史开发情况 | [todo.md](todo.md) |

## 必守规则

1. **版本号**：任何改动上线前，必须同步更新 `index.html` 里的 `APP_VERSION`（递增）和 `CHANGELOG`（新增一条放最前）。否则线上用户的自动版本检测不会触发强刷。
2. **部署**：用 `ws-quick-html-deploy` skill 的 `update-file` 命令更新线上页面，地址保持不变（本地已有部署历史记录，勿用 create 生成新地址）。
3. **上游同步**：本项目是 fork 性质的改造，`reference/upstream-bugs.md` 记录了对上游的 bug 修复；若上游更新需合并，先看该文档避免冲突。
4. **测试**：改动后至少做 JS 语法检查（提取 `<script>` 内容 `new Function()` 编译），涉及 UI 的改动用浏览器实测核心链路（加载视频 → 加批注 → 刷新 → 恢复）。
5. **单文件原则**：不要拆分 index.html（上游定位就是零依赖单文件，除非用户明确要求模块化）。
