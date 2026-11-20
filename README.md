# 墨水屏推图 · Web 版 v0.1

对应 APK `com.epaper.epaper_app 1.0.0`（Flutter，label 占位名 BlaBla）的网页版。
REA 分析见 `~/workspace/rea-analysis/apk-20261120/ANALYSIS.md`。

## 网上版（2026-11-20 已上线）

- **地址：https://zhuowater.github.io/epaper-web/**
- 托管：GitHub Pages，公开仓库 https://github.com/zhuowater/epaper-web （main 分支，index.html / sw.js / manifest.json / README.md 四个文件；本目录是源码，改完要重新 push 才会上线）。
- Vercel 曾尝试但失败：MCP 的 upload_file 报 403，连接无 `zhuowaters-projects` team 权限，且授权工具不在可用列表里——改走 GitHub Pages。
- **网上版能做什么**：任何地方打开，生图（Ark）、裁剪 1200×1600、灰度预览、下载 JPEG、历史（本源 IndexedDB）。
- **网上版不能直接推图**：https 页面访问 http://192.168.4.1 会被浏览器以混合内容拦截，页面上的推图按钮会直接提示并拒绝。流程是：网上版生图 → 下载 JPEG → 连板子 AP `ePaper-13in3` → 打开板子自带页面 `http://192.168.4.1` 选文件上传。直接推图仍需本页由板子同源 serve（或把这三个文件放进板子固件静态托管）。

## 文件（共 3 个，无构建步骤，板子直接 serve 即可）

- `index.html` — 全部功能（单文件，内联 CSS+JS）
- `sw.js` — Service Worker：同源缓存，让页面在离开板子 AP 后仍能打开
- `manifest.json` — PWA 清单（可安装到主屏幕）

## 功能对照 APK

| APK 流程 | 网页版 |
|---|---|
| 提示词（文字/语音 Paraformer） | 文字输入 + 浏览器语音识别（`webkitSpeechRecognition`，zh-CN）。Paraformer 需服务端 WS 中继，v0.1 未做 |
| 火山 Ark 生图 `doubao-seedream-5-0-flash-260915` | 直连 `ark.cn-beijing.volces.com/api/v3/images/generations`，Key 只存本机 `localStorage` |
| 裁剪 1200×1600 | Canvas 居中裁剪 + 灰度预览，可下载 JPEG |
| 推图 `POST http://192.168.4.1/upload` | 见下「推图形态探测」 |
| 查状态 `GET /status` | 「查 /status」按钮，原样显示响应 |
| 历史记录 | IndexedDB 存裁剪后的图（断网/切 WiFi 后仍在） |
| WiFi 指示 / 按网络路由 | 网页做不到：需人手动连 AP `ePaper-13in3` |

## 部署形态一（推荐，板子 serve）

把 3 个文件放到板子固件的静态文件里，由板子在 `http://192.168.4.1/` serve。
使用流程：

1. 手机流量下打开本页（第一次需在 AP 内打开一次，让 SW 缓存住）→ 输入提示词 → AI 生图 → 点「存入历史」
2. 连板子 AP `ePaper-13in3` → 打开 `http://192.168.4.1/` → 历史里取出图片 → 「推图到板子」
3. 全程同源，没有混合内容、没有 CORS 问题。

## 部署形态二（家里局域网 + 小后端）

局域网 HTTP 托管本页 + 一个小后端代理 Ark 接口（Key 放后端，不进浏览器，
也绕过 Ark 的 CORS）。推图仍需与板子同源或后端转发 `/upload`，否则浏览器
跨域拦截。Key 不进浏览器是这种形态的唯一好处；v0.1 未附后端代码，需要再说。

## 推图形态探测（未确认项的解法）

静态分析未能确认 `/upload` 的 payload 形态（multipart 字段名 vs 原始 JPEG body），
固件源码是最终权威。页面按以下顺序自动尝试，并在日志里写明哪一种成功：

1. 原始 JPEG body（`Content-Type: image/jpeg`）——ESP32 类板端最常见
2. multipart，字段名 `file`
3. multipart，字段名 `image`

**2026-11-20 补充（静态证据，尚未上板验证）：** 对 APK 的 `libapp.so`（Dart AOT，6.5MB）
再做了一次字符串级核查——全文件 0 处出现 `multipart`，`package:http/src/multipart_request.dart`
未被链接进来（同包的 request/client/base_client 等都在），也没有 dio；
而 `image/jpeg` 存在。Dart AOT 会摇掉未用到的库文件，所以 App 几乎肯定没用
multipart，而是 `http.post` 直接发原始 JPEG body。**上板时第一档（原始 JPEG）大概率
直接成功**；页面仍保留三档探测作兜底，若第一档意外失败，那不是字段名问题，
请把日志发回来看。第一次在真板子上成功后，把成功的形态告诉 Spark，固定为默认，去掉多余尝试。

**2026-11-20 固件确认：** jacky 发来板子源码 `volc_epaper_13in3`（已解包到
`~/workspace/epaper-firmware/volc_epaper_13in3/`）。`config.h`：热点 SSID
`ePaper-13in3`、密码 `12345678`、网关 `192.168.4.1`。`.ino` 注释写明
"APP/网页 POST image/jpeg 必须用 server.raw() 收字节；用 upload() 会空指针崩溃"，
板载网页本身也是 `fetch("/upload", {headers: {"Content-Type": "image/jpeg"}, body: picked})`。
**`/upload` = 原始 JPEG body 已由固件源码证实**，不再是推测；三档探测保留作兜底即可。
另外板子自带一个极简上传页（选文件→上传并刷屏），网页版 v0.1 的增量是
Ark 生图、裁剪 1200×1600、灰度预览、历史记录和语音输入。

## 已知限制（诚实标注）

- **未在真板子上实测**：沙箱连不到 `192.168.4.1`。JS 语法已过 `node --check`，
  裁剪/推图逻辑已用本地 mock 板端跑通（见下），但 `/upload` 的真实形态、
  `/status` 的响应字段、Ark 的浏览器 CORS 表现，都要第一次上板验证。
- Ark 直连可能被 CORS 拦截（原生 App 无此问题）。若生图失败，页面会明说；
  退路是「选本地图片」或形态二的后端代理。
- 语音识别依赖浏览器实现，效果不如 Paraformer；不做实时转写。

## 本地验证记录（2026-11-20）

- `node --check`：`index.html` 内联脚本语法通过。
- 推图 fallback：抽取页面里的 `uploadBlob` 原代码，用 Node 直连本地 mock 板端
  实测三档——只接受原始 JPEG / 只接受 multipart `file` / 全部拒绝——
  均按预期：成功形态被正确识别并记录，失败时三档都试过才报错。
- 未验证：真板子 `/upload`、`/status` 字段、Ark CORS、手机端 PWA 安装、
  Canvas 裁剪（仅代码复核，未在浏览器里跑过）。
