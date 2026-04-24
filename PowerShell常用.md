### 配置 Profile 文件
Profile 就是 PowerShell 启动时自动加载的配置脚本

```powershell
# 查看 profile 文件路径
$PROFILE

# 查看是否已存在
Test-Path $PROFILE

# 没有就创建
New-Item -Path $PROFILE -Type File -Force
```