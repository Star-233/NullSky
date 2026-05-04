## PHP 反引号 `` ` ``

在 PHP 的官方文档中，反引号属于**运算符**的一种。它的作用是：**将其括起来的内容作为 shell 命令执行，并将执行结果以字符串的形式返回。**

*   **语法格式：** `` `command` ``
*   **等效函数：** 它的功能与 PHP 内置函数 `shell_exec()` **完全相同**。

## PHP `system()`
执行shell命令

## PHP `eval()`
执行php代码

## PHP `assert()`
原本是拿来下条件断点的
- 在 PHP 7.x 及更早版本中，如果传入字符串，它会像 `eval()` 一样执行代码。
- 在 PHP 8.0 中，`assert()` 不再支持代码执行。

## PHP `passthru()`

passthru() 执行一个外部命令，并不对输出进行任何拦截或处理，而是直接将命令运行的结果（通常是二进制数据或大量文本）传递给输出。
- 不会返回字符串，而是直接输出执行的原始完整结果

## PHP `inlcude()`
当传入 `include()` 的是 PHP 代码的时候，`include()` 将会执行他，反之，如果传入的是是一段朴素的文本，那么会在页面上直接显示他。

`data://` 是 PHP 的伪协议，同时，他遵循 [RFC 2397](https://datatracker.ietf.org/doc/html/rfc2397) 标准。 
传统来说，一般链接存放的是文件的地址，而 `data://` 是直接把文件内容本身嵌入进链接里面。

我们可以通过 `include()` 和 `data://` 配合来实现 RCE
例如 `include("data://text/plain,<?php system('ls')?>")`
或者使用base64编码 `include("data://text/plain;base64,PD9waHAgc3lzdGVtKCd3aG9hbWknKTs/Pg==")`