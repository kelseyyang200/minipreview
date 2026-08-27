# 导入 / 导出模块（普通 + 超级）

## 功能概述

顶部「📦 导入/导出 ▼」菜单（1.4.0+），六个菜单项分两组：

| 组 | 菜单项 | 函数 | 说明 |
| --- | --- | --- | --- |
| 导出 | 📝 文本格式 | `exportAsText()` | 原有功能 |
| 导出 | 🖼️ 图片格式 | `exportAsImage()` | 原有功能 |
| 导出 | 💾 批注文件（很小的 JSON） | `exportAnnotationsFile()` | **普通导出** |
| 导出 | 📦 超级导出 | `superExport()` | **批注 + 原视频打包 ZIP** |
| 导入 | 📂 批注文件导入 | `importAnnotationsFile(event)` | **普通导入**（`#importJsonInput`） |
| 导入 | 📦 超级导入 | `superImport(event)` | **完整包还原**（`#importZipInput`） |

## 普通导入导出（批注 JSON，KB 级小文件）

导出格式（`批注_<项目名>_<日期>.json`）：

```json
{
  "type": "minipreview-annotations",
  "formatVersion": 1,
  "exportedAt": "ISO时间", "appVersion": "…",
  "project": { "name", "sourceType", "videoCount", "fileNames"(本地视频才有) },
  "annotations": [ /* 原样导出批注数组 */ ]
}
```

导入规则：**应用到当前已加载项目**（确认后替换其全部批注并落库）；未加载项目时提示先加载。批注的 videoIndex/videoName 与文件顺序对应，跨项目导入需自己保证文件一致。

## 超级导出（ZIP 完整包）

`manifest.json` + 视频文件本体（store 模式 ZIP，文件名 `项目名_完整包_日期.zip`）：

- 本地视频项目（files/dirfile/dirall）：`videos[]` 全部 File 打包；**重名文件加 `N_` 序号前缀**避免 ZIP 内冲突（fileNames 同步记录，导入后按新名建项目）。
- 链接类（url/wedrive）：无法打包本体，`manifest.project.url` 携带来源链接，导入时还原为链接项目。
- manifest 结构：`{ type:'minipreview-bundle', bundleVersion:1, project:{name, sourceType, url?, fileNames?, videoCount?}, annotations }`。

## 超级导入

`parseZip` 解析（仅支持 store）→ 找 manifest.json → 按类型还原：
- `fileNames`：逐个取视频构造 `new File([data], name)` → `loadVideoFiles(files, {handles:[]})`（确定性 id，文件名同则复用原项目缓存）→ 批注覆盖。
- `url`（无 fileNames）：url → `loadVideo + openProjectRecord`；wedrive → `enableWeDriveMode + openProjectRecord`。
- 最后统一 `annotations = manifest.annotations; renderAnnotations(); saveAnnotationsNow()`（覆盖该项目缓存里的旧批注）。

## 零依赖 ZIP 实现（重点算法）

- `CRC_TABLE` / `crc32OfBlob(blob)`：分块流式 CRC32（`blob.stream()` 逐块），大视频不整体进内存。
- `buildZip(entries)`：store 模式；local header(30B) + name(UTF-8, flags 0x0800) + blob 原样引用拼进 `Blob parts`；central directory + EOCD。**已用 Python `zipfile` 交叉验证**（CRC/中文名/结构全过）。
- `parseZip(file)`：尾部扫 EOCD → 遍历 central directory → 跳 local header 取 data（`method !== 0` 报错，只收自家包）。
- 边界：条目数 < 65535（u16）；包总大小 < 4GB（u32 偏移）。

## 常见改动点

- **支持压缩包导入**：需要引入 inflate（pako），违背零依赖原则，暂不做。
- **导出到文件夹**（不打包）：可改用 `showDirectoryPicker + createWritable` 逐个写文件，适合超大视频，见 sidebar-workspace.md 的句柄用法。
