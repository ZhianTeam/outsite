# 跨站网络加速

部署在 Cloudflare Workers 上的反向代理服务，通过将目标网站的请求转发到 Worker，实现对被限制网站的访问加速。内置动态 JS 钩子，自动将页面内的所有链接、请求重写为代理路径，保证页面内跳转和动态请求也能正常走代理。

---

## 工作原理

```
用户浏览器
    │
    ▼
Cloudflare Worker（你的域名）
    │  重写请求头 Host
    ▼
目标网站（如 www.google.com）
    │
    ▼
Worker 处理响应
    │  HTML：替换所有链接 + 注入 JS 钩子
    │  非 HTML：直接透传
    ▼
用户浏览器
```

---

## URL 格式

Worker 支持两种路径格式：

| 格式 | 示例 | 说明 |
|------|------|------|
| `/域名/路径` | `/www.google.com/search?q=test` | 默认使用 HTTPS |
| `/协议//域名/路径` | `/https//www.google.com/search` | 显式指定协议 |

访问根路径 `/` 返回内置的使用引导页面。

---

## 部署

### 前置条件

- Cloudflare 账号
- 一个绑定到 Cloudflare 的域名（可选，也可用 `*.workers.dev` 免费子域名）

### 步骤

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
2. 进入 **Workers & Pages** → **创建 Worker**
3. 将 `worker.js` 的全部内容粘贴到编辑器
4. 点击 **部署**
5. 访问分配的 `*.workers.dev` 地址，或绑定自定义域名

### 绑定自定义域名

Workers & Pages → 你的 Worker → **触发器** → **添加自定义域** → 填入域名 → 保存。

---

## 使用方式

### 网页界面

直接访问 Worker 根路径，会显示内置的引导页面，输入目标域名点击「立即访问」即可。

### 直接构造 URL

```
https://你的worker域名/www.google.com
https://你的worker域名/github.com/torvalds/linux
https://你的worker域名/https//example.com/path?query=1
```

---

## 技术细节

### HTML 响应处理

Worker 对 HTML 页面做以下处理：

1. `href/src/action` 中的绝对链接 → 重写为代理路径
2. `//host/path` 协议相对链接 → 重写为代理路径
3. CSS `url("https://...")` → 重写为代理路径
4. 以 `/` 开头的相对链接 → 补全为 `/目标域名/路径`
5. 在 `</body>` 前注入动态 JS 钩子

### JS 钩子劫持范围

注入的脚本会在页面运行时劫持以下 API，确保动态产生的请求也走代理：

- `window.fetch`
- `XMLHttpRequest.prototype.open`
- `window.open`
- `HTMLElement.prototype.setAttribute`（`src` / `href`）
- `Node.prototype.appendChild`
- `location.assign` / `location.replace` / `location.href`
- `history.pushState` / `history.replaceState`

内置防无限重定向保护（最多 5 次重定向后停止）。

### 非 HTML 响应

图片、JS、CSS、字体、JSON 等非 HTML 内容直接透传，不做任何修改。

---

## 限制与注意事项

- **WebSocket** 不支持（Cloudflare Workers 免费版限制）
- **部分网站** 有 Cloudflare 检测或 IP 封锁，可能无法正常代理
- **Service Worker / PWA** 类网站可能因 JS 沙箱限制无法完整运行
- Cloudflare Workers 免费版每天有 **10 万次请求**限额，超出后需升级付费计划
- 本工具仅供学习和合法用途，请遵守当地法律法规

---

## 文件说明

| 文件 | 说明 |
|------|------|
| `worker.js` | Worker 全部逻辑，单文件部署 |
