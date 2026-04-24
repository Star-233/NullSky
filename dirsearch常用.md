---
tags:
  - 实用
  - CTF
---
## 基础选项

| 选项              | 说明                      |
| --------------- | ----------------------- |
| `-u URL`        | 目标 URL                  |
| `-e EXTENSIONS` | 指定文件扩展名，如 `php,asp,txt` |

[[dirsearch 指定文件扩展名是如何工作的]]

```bash
dirsearch -u https://target.com -e php,asp,html
```

## 字典相关

| 选项             | 说明                             |
| -------------- | ------------------------------ |
| `-w WORDLISTS` | 指定字典文件或目录                      |
| `-f`           | 强制给每个字典条目都追加扩展名（默认只替换 `%EXT%`） |
| `--prefixes`   | 给条目加前缀                         |
| `--suffixes`   | 给条目加后缀                         |
```bash
# 指定字典文件
dirsearch -u https://target.com -w /usr/share/dirsearch/wordlists/common.txt
# 指定目录，会使用该目录的所有字典
dirsearch -u https://target.com -w /usr/share/dirsearch/wordlists/
# 混合指定
dirsearch -u https://target.com -w /path/to/custom.txt,/usr/share/dirsearch/wordlists/common.txt
```
## 过滤与排除

| 选项                   | 说明                            |
| -------------------- | ----------------------------- |
| `-i CODES`           | **只显示**指定状态码（如 `200,300-399`） |
| `-x CODES`           | **排除**指定状态码（如 `404,500-599`）  |
| `--exclude-sizes`    | 按响应大小排除（如 `0B,4KB`）           |
| `--exclude-text`     | 按响应内容排除                       |
| `--exclude-regex`    | 按正则排除响应                       |
| `--exclude-response` | 排除与某个页面相似的响应（如自定义 404 页面）     |
## 递归扫描

| 选项                   | 说明             |
| -------------------- | -------------- |
| `-r`                 | 递归扫描发现的目录      |
| `-R DEPTH`           | 最大递归深度         |
| `--recursion-status` | 递归时只对哪些状态码继续深入 |
| `--deep-recursive`   | 对每个层级都递归       |
## 请求定制

| 选项               | 说明                                   |
| ---------------- | ------------------------------------ |
| `-m METHOD`      | HTTP 方法（默认 GET）                      |
| `-d DATA`        | POST 数据                              |
| `-H HEADER`      | 自定义请求头（可多次使用）                        |
| `--cookie`       | 指定 Cookie                            |
| `--auth`         | 认证凭据（`user:password` 或 Bearer token） |
| `-F`             | 跟随重定向                                |
| `--random-agent` | 每次请求随机 User-Agent                    |
## 性能控制

| 选项           | 说明             |
| ------------ | -------------- |
| `-t THREADS` | 线程数（默认 30）     |
| `--timeout`  | 连接超时           |
| `--delay`    | 请求间隔           |
| `--max-rate` | 每秒最大请求数        |
| `-p PROXY`   | 代理（HTTP/SOCKS） |
## 输出

| 选项          | 说明                             |
| ----------- | ------------------------------ |
| `-o PATH`   | 输出到文件                          |
| `-O FORMAT` | 输出格式（`json,csv,md,html,xml` 等） |
| `-q`        | 安静模式，只输出结果                     |

---
通用场景
```bash
# 基础扫描，只看 200/301/302
dirsearch -u https://target.com -e php,html -i 200,301,302

# 带认证 + Cookie 的扫描
dirsearch -u https://target.com -e php --auth admin:pass --cookie "session=abc123"

# 递归扫描，排除自定义 404 页面
dirsearch -u https://target.com -e php -r --exclude-response 404.html

# 限速扫描
dirsearch -u https://target.com -e php --max-rate 50

# 结果导出为 JSON
dirsearch -u https://target.com -e php -o result.json -O json
```

针对Java
```bash
dirsearch -u https://target.com -e jsp,jspa,do,action,html,xml,properties,json
```