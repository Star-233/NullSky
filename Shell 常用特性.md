---
tags:
  - 实用
  - CTF
---
## 通配符

|   |   |   |   |
|---|---|---|---|
|`*`|匹配 **任意数量** 的任意字符（包括 0 个）|`ls *.txt`|所有 `.txt` 结尾的文件|
|`?`|匹配 **恰好 1 个** 任意字符|`ls file?.log`|`file1.log`、`fileA.log`（不匹配 `file10.log` 或 `file.log`）|
|`[abc]`|匹配括号内 **任意一个** 字符|`ls [abc].dat`|`a.dat`、`b.dat`、`c.dat`|
|`[a-z]` 或 `[0-9]`|匹配指定 **范围** 内的单个字符|`ls report[1-3].csv`|`report1.csv`、`report2.csv`、`report3.csv`|
|`[^...]` 或 `[!...]`|匹配 **不在括号内** 的单个字符|`ls [^0-9].txt`|不以数字开头的 `.txt` 文件|
