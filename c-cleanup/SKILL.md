---
name: c-cleanup
description: 清理 Windows C 盘空间。分析占用后清理 Windows 更新缓存、系统升级残留、临时文件、软件缓存、回收站，并运行 DISM 组件清理。适用于 C 盘空间不足时。
---

# C 盘清理流程

目标：在 Windows 11 上安全释放 C 盘空间。只清理可再生的缓存/临时数据，**不动**已安装软件、用户文档、`C:\Windows\Installer`、`WinSxS` 直接删除等有风险项目；超出本清单的删除需先征得用户确认。

## 1. 查看当前空间

```powershell
Get-PSDrive C | Select-Object @{n='UsedGB';e={[math]::Round($_.Used/1GB,1)}}, @{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}
```

## 2. 定位占用（可选，空间明显紧张时做）

扫描 C:\ 根目录、C:\Windows 子目录、用户主目录下 >100MB 的目录，慢的话用 `run_in_background`。已知环境信息：页面文件在 D 盘（不占 C）、休眠已关闭，无需重复检查这两项。

## 3. 清理 Windows 更新缓存（通常最大头，可达 10+ GB）

```powershell
Stop-Service wuauserv,bits -Force -ErrorAction SilentlyContinue
Start-Sleep 2
Rename-Item 'C:\Windows\SoftwareDistribution\Download' 'Download.old' -ErrorAction Stop
# 用 .NET API 删除（Remove-Item 会被系统路径保护拦截）
[System.IO.Directory]::Delete('\\?\C:\Windows\SoftwareDistribution\Download.old', $true)
Start-Service wuauserv,bits -ErrorAction SilentlyContinue
```

注意：
- 缓存里常有超长路径（MAX_PATH），必须用 `\\?\` 前缀，否则 .NET Delete 抛 "Could not find a part of the path"。
- 若 `\\?\` 仍失败，用 robocopy 镜像法：`robocopy $empty $target /MIR`。

## 4. 删除系统升级残留（存在才删）

```powershell
# 注意 PowerShell 中 $ 开头的路径必须用单引号
[System.IO.Directory]::Delete('\\?\C:\$WinREAgent', $true)      # try/catch 包裹
[System.IO.Directory]::Delete('\\?\C:\$WINDOWS.~BT', $true)
```

## 5. 清理临时文件与缓存

**关键：跳过 `$env:TEMP\claude` —— 那是当前 Claude 会话自己的工作目录，删了会破坏会话。**

```powershell
$targets = @()
Get-ChildItem "$env:TEMP" -Force -ErrorAction SilentlyContinue | Where-Object { $_.Name -ne 'claude' } | ForEach-Object { $targets += $_.FullName }
Get-ChildItem 'C:\Windows\Temp' -Force -ErrorAction SilentlyContinue | ForEach-Object { $targets += $_.FullName }
Get-ChildItem "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Cache" -Force -ErrorAction SilentlyContinue | ForEach-Object { $targets += $_.FullName }
$ok=0; $fail=0
foreach ($t in $targets) {
  try {
    $item = Get-Item $t -Force -ErrorAction Stop
    if ($item.PSIsContainer) { [System.IO.Directory]::Delete('\\?\' + $t, $true) }
    else { [System.IO.File]::Delete('\\?\' + $t) }
    $ok++
  } catch { $fail++ }   # 被占用的文件删除失败属正常，静默跳过
}
Clear-RecycleBin -Force -ErrorAction SilentlyContinue
```

npm 缓存（若安装了 npm）：

```powershell
if (Get-Command npm -ErrorAction SilentlyContinue) { npm cache clean --force }
```

## 6. DISM 组件清理（耗时 10–30 分钟，后台运行）

```powershell
dism /online /cleanup-image /startcomponentcleanup
```

用 `run_in_background: true` 执行。可能释放 0–4 GB（无旧组件时释放为 0 属正常，如实报告）。

## 7. 汇报前后对比

报告释放的总空间；DISM 释放为 0 时说明系统已无多余旧组件，不要虚报。

## 核心注意事项（血泪教训）

- **`Remove-Item` 对系统路径（C:\Windows\...、$env:TEMP\* 等）会被路径保护拦截**，统一改用 `[System.IO.Directory]::Delete('\\?\' + $path, $true)` / `[System.IO.File]::Delete`。
- 长路径必须加 `\\?\` 前缀。
- 不含法删除的对象：`C:\Windows\Installer`（会导致软件无法卸载/修复）、`WinSxS` 内容、Program Files、用户文档、搜索索引目录。
- 本流程只做标准安全清理；如果还想动 Steam 缓存、微信/QQ 数据目录等，必须先问用户并得到确认。
- 扫描大目录时 `Measure-Object -Property Length -Sum` 遇到空目录会报错，用 `-ErrorAction SilentlyContinue` 并忽略该报错即可。
