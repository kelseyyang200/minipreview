---
name: minipreview-dev
description: "开发或修改 MiniPreview 视频审片工具（本目录下的纯静态单文件项目 index.html）时使用。包括：新增/修改功能、修 bug、改样式、更新版本号、部署上线、给上游提 PR 前的整理。凡涉及本项目的任何代码改动都应先读本 Skill。"
version: 1.5.0
---

# MiniPreview 项目开发 Skill

本项目的所有代码改动、版本发布、部署都必须遵循本 Skill。

## 第一步：读开发文档（必做）

**先读 [docs/agent.md](../../docs/agent.md)**（约 2 分钟），它是开发入口，包含：

- 项目一句话简介与线上地址
- 目录管理表：要改哪一块 → 看哪份模块文档（渐进式披露，不要通读 index.html）
- 必守规则（版本号 / 部署 / 上游同步 / 测试）

按目录管理表跳转：

| 模块 | 文档 |
| --- | --- |
| 整体架构 / 数据流 / 存储设计 | docs/architecture.md |
| 侧边栏 + 工作目录 | docs/reference/sidebar-workspace.md |
| 最近打开 + 批注缓存 | docs/reference/recent-projects.md |
| 版本检测 / 强刷 / 更新日志 | docs/reference/version-refresh.md |
| 上游 bug 修复 / PR 拆分 | docs/reference/upstream-bugs.md |
| 待办 / 路线图 | docs/todo.md |

（路径相对于项目根目录，即本 Skill 所在仓库的根。）

## 硬性规则

### 1. 版本号（每次改动上线必做）

改动部署前，在 `index.html` 中同步更新两处：

```js
const APP_VERSION = '1.x.x';   // 递增
const CHANGELOG = [
  { version: '1.x.x', date: 'YYYY-MM-DD', items: ['改了什么', ...] },  // 新条目放最前
  ...
];
```

不更新版本号 = 线上用户的自动版本检测不会触发强刷 = 用户永远看不到新版本。更新后把 `docs/todo.md` 的开发历史表也加一行。

### 2. 部署（保持地址不变）

用 `ws-quick-html-deploy` skill（位于 `E:\共享\常用skills\ws-quick-html-deploy`）：

```
python "<skill路径>/scripts/html_deploy.py" update-file --path <项目根>/index.html --confirmed
```

本地已有部署历史记录，update-file 会复用地址，**不要用 create 生成新地址**。
线上地址：https://help.wshoto.com/resource/6fe1735d177343599a25c1f02a4c6d66/d94dca03.html

### 3. 测试（改动后必做）

```js
// JS 语法检查：提取 <script> 内容编译
new Function(html.match(/<script>([\s\S]*?)<\/script>/)[1]);
```

涉及 UI 的改动用浏览器实测核心链路：加载视频 → 加批注 → 刷新页面 → 批注恢复。

### 4. 单文件原则

所有 CSS/JS 内联在 `index.html`，不拆文件、不引构建工具、不加外部依赖（上游定位是零依赖单文件）。

### 5. 文档同步

新功能上线后在对应 `docs/reference/` 文档补一节（函数表 / 设计要点 / 已知边界），保持文档与代码一致，方便下一个 Agent 接手。
