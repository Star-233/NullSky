## 使用 `ctype`
### 单个变量模拟

| **C 类型**        | **ctypes 类型** | **备注**           |
| --------------- | ------------- | ---------------- |
| `int`           | `c_int`       | 32位有符号           |
| `unsigned int`  | `c_uint`      | 32位无符号           |
| `short`         | `c_short`     | 16位有符号           |
| `char`          | `c_char`      | 单个字节/字符          |
| `unsigned char` | `c_ubyte`     | 常用于处理字节流 (0-255) |
| `long long`     | `c_longlong`  | 64位有符号           |
| `float`         | `c_float`     | 单精度浮点            |
| `double`        | `c_double`    | 双精度浮点            |
| `char *`        | `c_char_p`    | 字符串指针（以 `\0` 结尾） |
| `void *`        | `c_void_p`    | 通用指针             |

```python
from ctypes import *

# 整数类型
a = c_int32(0x7FFFFFFF)
print(f"初始值: {a.value}")
a.value += 1
print(f"溢出后的值: {a.value}") # 输出 -2147483648

# 字符类型
char_val = c_char(b'A')
print(f"字符: {char_val.value}")

# 无符号字节
byte_val = c_ubyte(255)
byte_val.value += 1
print(f"字节溢出: {byte_val.value}") # 输出 0
```

### 数组模拟
在 C 中，数组定义为 `int arr[10]`。在 `ctypes` 中，定义数组的语法是：`类型 * 长度`。

```python
from ctypes import *
# 定义一个长度为 8 的 int 数组类型
IntArray8 = c_int * 8

# 实例化数组
v17 = IntArray8(80, 64227, 226312059, -1540056586, 20512, 3833, 0x56A9, 0)

# 像 Python 列表一样访问和遍历
print(f"v17[2] 的值: {v17[2]}")

print("遍历数组:")
for x in v17:
    print(hex(x), end=" ")
```

## 手动取 `&`
