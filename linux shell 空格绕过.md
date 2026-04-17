---
tags:
  - linux
---

## 使用 `${IFS}` 
`IFS` (Internal Field Separator) 是 Shell 的内部字段分隔符，默认值就是空格、制表符和换行符

常见的用法是 `${IFS}`
- 这里的 `{}` 是变量界定符，在 Shell 中，+只是个字符而不是连接符。例如你有一个变量 `name="Tom"`，然后你想输出 `"TomIsCool"` ，那么你需要 `echo ${IFS}IsCool`
	- 注意这并不是大括号扩展 (Brace Expansion)
- 实际的例子：`cat${IFS}/flag`

如果 `{}` 被禁用了，可以使用 `$9` 来分割，这是第九个命令参数，通常为空
- 实际的例子： `cat$IFS$9/flag`

如果 `$IFS` 后面跟着的不是字母和数字，也可以直接用
- 实际的例子：`cat$IFS/flag`

## 使用制表符（Tab）

实际的例子： `cat%09/flag.txt`
	这里 `%09` 是制表符

## 使用重定向符号 `<` `<>`
大多数使用了标准输入输出的都能用此方法绕过
- < (输入重定向)
	- `cat</flag`
	- `php<file`
- `<>` (读写重定向)
	- `cat<>/flag`
