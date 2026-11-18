# Xianyu Login State Extractor（闲鱼登录态提取 Chrome 扩展）

这是一个基于 **Manifest V3**（版本 `1.1`）的浏览器扩展，用于从已登录的闲鱼网页（`goofish.com`）提取完整的 Cookie（含 `HttpOnly`）及当前浏览器环境指纹，供 AI Goofish Monitor 爬虫任务加载使用。

---

## 安装方法

### 方式一：本地加载源码（开发者模式）

1. 打开 Chrome 或 Edge 浏览器，访问 `chrome://extensions`（Edge 可访问 `edge://extensions`）。
2. 打开右上角的 **开发者模式（Developer mode）** 开关。
3. 点击左上角 **加载已解压的扩展程序（Load unpacked）**。
4. 选择本仓库下的 `chrome-extension` 目录，扩展图标即会出现在浏览器工具栏中。

### 方式二：Chrome 应用商店安装

- 访问 [Chrome Web Store - Xianyu Login State Extractor](https://chromewebstore.google.com/detail/xianyu-login-state-extrac/eidlpfjiodpigmfcahkmlenhppfklcoa) 直接安装。

---

## 使用步骤

1. 在浏览器当前标签页打开 [https://www.goofish.com](https://www.goofish.com) 并登录你的闲鱼账号。
2. 保持该 `goofish.com` 标签页处于激活状态，点击浏览器工具栏中的扩展图标打开弹窗。
3. 点击按钮 **`1.点击获取环境+登录状态`**：
   - 按钮会变为 `采集中，请稍候...`，等待约 1～2 秒，下方文本框会展示完整 JSON，并提示 `采集完成，已生成JSON`。
4. 点击按钮 **`2.点击复制`**，将生成的 JSON 复制到剪贴板。
5. 导入到监控系统（任选其一）：
   - **Web UI 多账号管理（推荐）**：打开监控后台 -> **账号管理** -> **添加账号**，填写账号名称（仅限字母、数字、下划线、短横线）并粘贴 JSON，系统会保存为 `state/<账号名>.json`。
   - **手动保存文件**：将 JSON 保存到仓库 `state/<账号名>.json`，或保存为根目录默认状态文件 `xianyu_state.json`（也可调用 `POST /api/login-state` 更新）。

---

## 导出的 JSON 结构与采集原理

扩展声明了 `activeTab`、`cookies`、`scripting`、`storage`、`tabs`、`webRequest` 权限以及 `*://*.goofish.com/*` 主机权限，导出的 JSON 顶层字段如下：

```json
{
  "capturedAt": "2026-03-19T08:00:00.000Z",
  "pageUrl": "https://www.goofish.com/",
  "page": {
    "pageUrl": "https://www.goofish.com/",
    "referrer": null,
    "visibilityState": "visible"
  },
  "env": {
    "navigator": { "userAgent": "...", "platform": "...", "language": "zh-CN", "languages": ["zh-CN"], "hardwareConcurrency": 8, "deviceMemory": 8, "maxTouchPoints": 0, "webdriver": false, "doNotTrack": null, "userAgentData": {} },
    "screen": { "width": 1920, "height": 1080, "devicePixelRatio": 2, "colorDepth": 24 },
    "intl": { "timeZone": "Asia/Shanghai", "locale": "zh-CN" }
  },
  "storage": {
    "local": {},
    "session": {}
  },
  "meta": {
    "droppedStorageKeys": {
      "local": [],
      "session": []
    }
  },
  "headers": {
    "User-Agent": "...",
    "Accept-Language": "...",
    "sec-ch-ua": "..."
  },
  "cookies": [
    {
      "name": "...",
      "value": "...",
      "domain": ".goofish.com",
      "path": "/",
      "expires": 1770000000,
      "httpOnly": true,
      "secure": true,
      "sameSite": "None"
    }
  ]
}
```

- **Cookie 提取**：调用 `chrome.cookies.getAll({ url: origin })` 获取当前域名下包括 `HttpOnly` 在内的全部 Cookie，并将 Chrome 的 `sameSite` 枚举映射为 Playwright 兼容的 `"None" | "Lax" | "Strict"`。
- **页面环境与存储**：通过 `chrome.scripting.executeScript` 读取 `navigator`、`screen`、`Intl` 时区信息以及 `localStorage` / `sessionStorage`；单条长度超过 `4096` 字符的存储项会被自动剔除，并记录到 `meta.droppedStorageKeys`，防止快照体积过大。
- **请求头捕获**：临时注册 `chrome.webRequest.onBeforeSendHeaders`（含 `extraHeaders`，最长等待 2000ms），并在页面内触发一次轻量探测请求 `/__codex_probe?ts=...` 捕获真实请求头，按 17 个常见头部白名单过滤输出。
- **后端如何消费**：`src/scraper.py` 加载该 JSON 时会自动识别增强快照字段，将 UA、语言、时区、屏幕尺寸、DPR、触摸点同步到 Playwright `BrowserContext`，同时剥离 `Sec-Fetch-*`、`Accept`、`Cookie` 等应由浏览器按请求类型动态生成的头部，避免页面资源变成空 CORS 请求导致白屏。

---

## 常见问题排查

- **提示 `采集失败: 请先打开 goofish.com 页面` 或 `未找到活动标签页`**
  - 请确认当前浏览器窗口正处于 `https://www.goofish.com` 页面（不能停留在 `chrome://extensions`、监控后台页面或其他空白标签页）。
- **提示 `采集失败: 无法获取页面信息` 或 `通信失败`**
  - 可能是刚安装/重载扩展但当前闲鱼页面尚未刷新。先按 `F5` 刷新一次 `goofish.com` 页面，若在 `chrome://extensions` 修改过代码请先点击扩展卡片上的“刷新”按钮再重试。
- **导入后运行任务仍跳转登录页（`passport.goofish.com` / `mini_login`）**
  - 说明采集时网页实际未处于登录状态，或账号已在网页端退出/过期。请先在 `goofish.com` 确认右上角显示已登录个人昵称，重新提取并覆盖更新对应账号 JSON。
- **点击 `2.点击复制` 提示没有可复制的数据**
  - 需先点击 `1.点击获取环境+登录状态` 等待文本框生成 JSON 后再复制。
