
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

PHP 序列化后的字符串是一串具有特定规则的文本，用于描述变量的**类型**和**结构**。它的基本设计思想是：**“类型标识符 : 数据内容”**，不同类型的数据有不同的固定格式。

以下是 PHP 序列化字符串中各个数据类型的结构详解：

---

### 一、 基础数据类型

| 数据类型 | 格式 | 示例 | 解释 |
| :--- | :--- | :--- | :--- |
| **Null** | `N;` | `N;` | 无值 |
| **Boolean** | `b:<0或1>;` | `b:1;` | `true` 为 1，`false` 为 0 |
| **Integer** | `i:<数值>;` | `i:123;` | 整数 123 |
| **Double / Float** | `d:<数值>;` | `d:12.34;` | 浮点数 12.34 |
| **String** | `s:<长度>:"<字符串内容>";` | `s:5:"hello";` | 长度为 5 的字符串 "hello" |

---

### 二、 复合数据类型（数组与对象）

#### 1. 数组 (Array)
数组的结构包含**元素个数**，以及大括号 `{}` 内的**键值对**（键和值也都是序列化后的格式）。
*   **结构**：`a:<元素个数>:{键1;值1;键2;值2;...}`
*   **示例**：
    ```php
    // 原数组：[0 => "apple", "color" => "red"]
    a:2:{i:0;s:5:"apple";s:5:"color";s:3:"red";}
    ```

#### 2. 对象 (Object)
对象的结构包含**类名长度**、**类名**、**属性个数**，以及大括号 `{}` 内的**成员属性名与属性值**。
*   **结构**：`O:<类名长度>:"<类名>":<属性个数>:{属性名1(序列化);属性值1(序列化);...}`
*   **示例**：
    ```php
    // 假设有一个类名为 User 的对象，含有一个公有属性 $name = "admin"
    O:4:"User":1:{s:4:"name";s:5:"admin";}
    ```

---

### 三、 对象属性可见性对结构的影响（重要）

在对象序列化中，**属性的可见性（Public/Protected/Private）会直接改变属性名（Key）的序列化字符串和长度**。这是安全领域（如反序列化漏洞）中需要特别注意的细节。

#### 1. Public（公有属性）
*   **格式**：`属性名`
*   **示例**：`public $name;` -> 序列化后为 `s:4:"name";`

#### 2. Protected（受保护属性）
*   **格式**：`\x00*\x00属性名`（其中 `\x00` 代表空字节 Null Byte）
*   **规则**：属性名前会加上 `\x00*\x00`。因为有两个空字节和一个星号，所以**长度会在实际属性名长度的基础上加 3**。
*   **示例**：`protected $name;` -> 序列化后为 `s:7:"\x00*\x00name";`（`name`长度4，加上`\x00*\x00`长度3，总长7）

#### 3. Private（私有属性）
*   **格式**：`\x00类名\x00属性名`（其中 `\x00` 代表空字节 Null Byte）
*   **规则**：属性名前会加上 `\x00类名\x00`。所以**长度会在（属性名长度 + 类名长度）的基础上加 2**。
*   **示例**：在 `User` 类中定义 `private $name;` -> 序列化后为 `s:10:"\x00User\x00name";`（`User`长度4 + `name`长度4 + 两个空字节长度2 = 10）

---

### 四、 完整示例解析

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

### WAF绕过
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

- 字符串和数字做松散比较的时候，字符串会转换为数字
例如
```php
//（截取开头的数字，直到遇到非数字）
echo("877.php" == 0x36d);
// Output:1 
```
这是成立的

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

