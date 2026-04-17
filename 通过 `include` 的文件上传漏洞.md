---
tags:
  - CTF
  - 题目
---


[题目链接](https://ctf.bugku.com/challenges/detail/id/106.html)

## 过程

打开网页，没有明显的东西。此时需要寻找提示，查看源代码发现 `upload.php` 提示。再查看网页的url路径，推测是 `include`。
尝试使用php伪协议读取源代码，但被过滤了，没法读取。

尝试从`upload.php` 来入手，只能上传图片文件，不过，如果上传图片文件然后让服务器 `include` 的话，就可以执行代码。
首先使用下面的payload来验证一下：

```php
<php? system("ls /"); ?>
```

上传上去，然后url路径改为 `index.php?file=upload/<your-file>`
访问之后发现php标签被过滤掉了，php旧版本还有另一种标签形式，将payload改成：

```php
<script language=php>system("ls /");</script>
```

访问之后，打印出了根目录的内容，漏洞是存在的。
只需要 `system("cat /flag")` 就可以拿到flag了。

## 总结

`include` 不管你的后缀是不是 `.php` ，只要里面有代码就会执行。
