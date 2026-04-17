---
tags:
  - CTF
---
## 使用注释符号绕过
```SQL
-- 原始语句
UNION SELECT 1, 2, 3
-- 绕过语句
UNION/**/SELECT/**/1,2,3
```