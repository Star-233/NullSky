
## 安装

[官方安装教程](https://pwndbg.re/stable/setup/#i-want-to-use-my-system-gdb-lldb)

## 和 pwntools 配合


> [!attention] 前置条件
> - WSL2
> - gnome-terminal
> - `curl --proto '=https' --tlsv1.2 -LsSf 'https://install.pwndbg.re' | sh -s -- -t pwndbg-gdb`


```python
context.terminal = ['gnome-terminal', '--', 'sh', '-c']
context.gdb_binary = 'pwndbg'

# gdb.attach(p, gdbscript=gs)
```

## 常用命令
- `stepover` - 在当前指令之后的指令处断点。
	- `so`
- 