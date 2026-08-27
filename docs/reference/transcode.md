# 一键转码模块（本机转码助手）

## 功能概述

浏览器解不出某视频画面时（典型：H.265/HEVC，`checkDecodability` 检出 `videoWidth === 0`），视频区顶部黄框警告里出现「🔧 一键转码为 H.264」按钮。点击后把视频发给**本机运行的转码助手**（Python + ffmpeg），转完自动加载新文件，全程不离开页面。

```
页面(https) ──ping/transcode/progress/result──> 本机助手 http://127.0.0.1:17832 ──> ffmpeg
```

## 页面侧（index.html）

| 函数 | 职责 |
| --- | --- |
| `checkDecodability()` | loadedmetadata 时 `videoWidth===0` → 显示 `#codecWarning`（含转码按钮） |
| `transcodeCurrentVideo()` | 状态机：ping(3s 超时) → POST FormData 上传（带 X-Video-Duration 头）→ 每秒轮询 progress → fetch result → `loadVideoFiles` 自动加载 `原名_h264.mp4` |
| `downloadHelperZip()` | 助手未运行时引导下载自助包（内嵌 py+bat+说明，用 buildZip 打包） |

状态文案显示在 `#transcodeStatus`；助手不在线 → confirm 引导下载包；`ffmpeg: false` → 提示 `winget install Gyan.FFmpeg`。

## 助手侧（mp_transcode_helper.py）

- 位置：`C:\Users\Administrator\mp-transcode-helper\`，双击「启动转码助手.bat」运行（窗口保持开启）。
- 接口：`GET /ping`、`POST /transcode`（multipart + X-Video-Duration）、`GET /progress/<job>`（percent 按 ffmpeg stderr 的 time=/duration 估算）、`GET /result/<job>`（octet-stream，送达后清理临时文件）。
- 编码策略（attempts 依次回退）：`h264_nvenc -cq 20`（NVIDIA 硬编，快数倍）→ `libx264 -crf 18 -preset fast`；音频先 `copy` 失败再 `aac`。
- CORS：`Access-Control-Allow-*` + `Access-Control-Allow-Private-Network: true`（Chrome 对 https→127.0.0.1 的 preflight 要求），OPTIONS 已处理。
- 只监听 127.0.0.1，不对外网开放；文件不出本机。

## ⚠️ 双副本同步（重要）

`HELPER_PY`（页面内嵌，供自助下载）与 `C:\Users\Administrator\mp-transcode-helper\mp_transcode_helper.py` 必须保持一致。**改助手代码时**：改本地文件 → 用 node 脚本同步回页面。

**同步脚本注意**：不能用 `String.replace(pattern, replacementString)`——Python 源码里的 `$'`（如 `re.sub(r'\.[^.]+$', ...)`）会被当作 replace 特殊模式破坏内容。必须用**函数替换或字符串拼接**：

```js
const local = fs.readFileSync(pyPath, 'utf8').replace(/\r\n/g, '\n').trim();
const s = 'const HELPER_PY = String.raw`';
const i = html.indexOf(s), j = html.indexOf('const HELPER_BAT', i);
html = html.slice(0, i) + s + local + '`;\n\n        ' + html.slice(j);
```

（此坑已实际踩过：replace 字符串模式导致模板字符串未闭合、整页 JS 语法错误。）

## 已知边界

- 转码速度：软编约 1.5~2x 实时（10 分钟视频约 5~7 分钟）；NVENC 可用时可到 5x+。
- 链接类项目（URL/微盘）无法一键转码（拿不到文件本体），按钮会提示。
- 助手单线程单任务，转码中再点会排队（JOBS 字典 + 线程）。
