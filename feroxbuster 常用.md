## 🔹 基础选项

|参数|说明|
|---|---|
|`-h, --help`|打印帮助信息|
|`-V, --version`|打印版本信息|

---

## 🔹 目标选择 (Target selection)

|参数|说明|
|---|---|
|`-u, --url <URL>`|目标 URL（必需，除非使用 `--stdin`/`--resume-from`/`--request-file`）|
|`--stdin`|从标准输入读取 URL 列表|
|`--resume-from <STATE_FILE>`|从中断的状态文件恢复扫描|
|`--request-file <REQUEST_FILE>`|使用原始 HTTP 请求文件作为模板|

---

## 🔹 复合设置 (Composite settings)

|参数|说明|
|---|---|
|`--burp`|自动设置 proxy 为 `http://127.0.0.1:8080` 并启用 `--insecure`|
|`--burp-replay`|设置 replay-proxy 为 Burp 并启用 `--insecure`|
|`--data-urlencoded <DATA>`|设置 POST 表单数据 + 相应 Content-Type|
|`--data-json <DATA>`|设置 POST JSON 数据 + 相应 Content-Type|
|`--smart`|启用 `--auto-tune` + `--collect-words` + `--collect-backups`|
|`--thorough`|在 `--smart` 基础上增加 `--collect-extensions` + `--scan-dir-listings`|

---

## 🔹 代理设置 (Proxy settings)

|参数|说明|
|---|---|
|`-p, --proxy <PROXY>`|设置代理（支持 http/https/socks5）|
|`-P, --replay-proxy <REPLAY_PROXY>`|仅将未过滤的请求发送到重放代理|
|`-R, --replay-codes <CODE>...`|指定哪些状态码的响应发送到 replay-proxy|

---

## 🔹 请求设置 (Request settings)

|参数|说明|
|---|---|
|`-a, --user-agent <UA>`|设置 User-Agent|
|`-A, --random-agent`|使用随机 User-Agent|
|`-x, --extensions <EXT>...`|指定要扫描的文件扩展名（支持 `@file` 读取）|
|`-m, --methods <METHOD>...`|指定 HTTP 方法（默认 GET）|
|`--data <DATA>`|请求体数据（支持 `@file`）|
|`-H, --headers <HEADER>...`|添加自定义 HTTP 头|
|`-b, --cookies <COOKIE>...`|添加 Cookie|
|`-Q, --query <QUERY>...`|添加 URL 查询参数|
|`-f, --add-slash`|给每个 URL 末尾添加 `/`|
|`--protocol <PROTOCOL>`|指定协议（http/https，默认 https）|

---

## 🔹 请求过滤 (Request filters)

|参数|说明|
|---|---|
|`--dont-scan <URL>...`|排除不扫描的 URL 或正则pattern|
|`--scope <URL>...`|添加额外域名/URL 到扫描范围|

---

## 🔹 响应过滤 (Response filters) ⭐ 核心功能

|参数|说明|
|---|---|
|`-S, --filter-size <SIZE>...`|按响应体大小过滤|
|`-X, --filter-regex <REGEX>...`|按正则表达式过滤响应内容|
|`-W, --filter-words <WORDS>...`|按响应词数过滤|
|`-N, --filter-lines <LINES>...`|按响应行数过滤|
|`-C, --filter-status <CODE>...`|黑名单：排除指定状态码|
|`--filter-similar-to <URL>...`|过滤与指定页面相似的响应（用于软404）|
|`-s, --status-codes <CODE>...`|白名单：仅显示指定状态码（默认全部）|
|`--unique`|仅显示唯一响应|

---

## 🔹 客户端设置 (Client settings)

|参数|说明|
|---|---|
|`-T, --timeout <SECONDS>`|请求超时时间（默认 7 秒）|
|`-r, --redirects`|允许跟随重定向|
|`-k, --insecure`|禁用 TLS 证书验证|
|`--server-certs <PEM\|DER>...`|添加自定义根证书|
|`--client-cert/--client-key`|mTLS 双向认证用证书/密钥|

---

## 🔹 扫描设置 (Scan settings)

| 参数                              | 说明                    |
| ------------------------------- | --------------------- |
| `-t, --threads <NUM>`           | 并发线程数（默认 50）          |
| `-n, --no-recursion`            | 禁用递归扫描                |
| `-d, --depth <NUM>`             | 最大递归深度（默认 4，0 为无限）    |
| `--force-recursion`             | 强制对所有发现的路径进行递归        |
| `--dont-extract-links`          | 不从响应体中提取链接            |
| `-L, --scan-limit <NUM>`        | 限制并发扫描总数（0 为无限制）      |
| `--parallel <NUM>`              | 并行运行多个 feroxbuster 实例 |
| `--rate-limit <NUM>`            | 限制每秒请求数（每目录）          |
| `--response-size-limit <BYTES>` | 限制读取的响应体大小（默认 4MB）    |
| `--time-limit <TIME>`           | 限制总扫描时长（如 `10m`）      |
| `-w, --wordlist <FILE>`         | 指定字典文件路径或 URL         |
| `--auto-tune`                   | 错误过多时自动降速             |
| `--auto-bail`                   | 错误过多时自动停止             |
| `-D, --dont-filter`             | 禁用自动泛解析过滤             |
| `--scan-dir-listings`           | 强制递归扫描目录列表页面          |

---

## 🔹 动态收集设置 (Dynamic collection)

| 参数                            | 说明                           |
| ----------------------------- | ---------------------------- |
| `-E, --collect-extensions`    | 自动发现新扩展名并加入扫描                |
| `-B, --collect-backups`       | 自动请求常见备份后缀（如 `.bak`, `.old`） |
| `-g, --collect-words`         | 从响应中自动提取关键词加入字典              |
| `-I, --dont-collect <EXT>...` | 收集扩展名时排除指定类型                 |

---

## 🔹 输出设置 (Output settings)

|参数|说明|
|---|---|
|`-v, --verbosity...`|增加详细程度（`-vv` 更详细）|
|`--silent`|仅输出 URL，关闭日志（适合管道）|
|`-q, --quiet`|隐藏进度条和横幅|
|`--json`|以 JSON 格式输出日志|
|`-o, --output <FILE>`|结果输出文件|
|`--debug-log <FILE>`|调试日志输出文件|
|`--no-state`|禁用 `.state` 状态文件生成|
|`--limit-bars <NUM>`|限制同时显示的进度条数量|

---

## 🔹 更新设置

|参数|说明|
|---|---|
|`-U, --update`|更新到最新版本|

---

