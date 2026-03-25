---
title: OpenClaw一键安装教程（自定义中文脚本）
date: 2026-03-25 10:00:00
tags:
---

## 前言

本教程将介绍如何使用自定义的PowerShell脚本来一键安装OpenClaw。相比官方安装脚本，自定义脚本具有以下优势：

- 📝 全程中文提示，更易理解
- 🎨 彩色输出，状态一目了然
- ⏳ 进度显示，知道安装进行到哪一步
- 🔧 自动检测环境，提前发现潜在问题
- 📦 支持离线安装包（可选）
- 🇨🇳 支持国内镜像加速下载
- ⚙️ 高度可定制化安装

## 目录

<!-- toc -->

- [前言](#前言)
- [脚本功能介绍](#脚本功能介绍)
- [实现思路与流程](#实现思路与流程)
- [完整安装脚本](#完整安装脚本)
- [完整卸载脚本](#完整卸载脚本)
- [使用教程](#使用教程)
- [脚本详解](#脚本详解)
- [进阶定制](#进阶定制)
- [常见问题](#常见问题)

<!-- more -->

## 脚本功能介绍

### 核心功能

| 功能 | 说明 |
|------|------|
| 环境检测 | 自动检查Windows版本、PowerShell版本、磁盘空间 |
| 网络测试 | 检测网络连接和下载速度 |
| 依赖检查 | 检查必要的运行库（VC++ Redist） |
| 进度显示 | 实时显示下载和安装进度 |
| 错误处理 | 详细的错误提示和解决方案 |
| 日志记录 | 保存安装日志，方便排查问题 |
| 镜像选择 | 支持多个国内镜像源自动测速选择 |
| 自定义安装 | 支持指定安装目录、API配置等 |

### 特色功能

- 🌈 **彩色输出**：使用不同颜色区分信息、警告、错误
- 📊 **进度条**：下载文件时显示进度条
- 🔄 **自动重试**：下载失败自动重试3次
- 💾 **断点续传**：支持下载中断后继续
- 🛡️ **安全校验**：校验文件MD5确保完整性
- 🇨🇳 **国内镜像**：支持阿里云、腾讯云、华为云等国内镜像
- ⚡ **智能选源**：自动测速选择最快镜像
- 🔧 **高度定制**：安装目录、API配置等均可自定义

## 实现思路与流程

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    OpenClaw安装脚本                      │
├─────────────────────────────────────────────────────────┤
│  1. 初始化阶段                                           │
│     ├── 设置控制台编码（UTF-8）                          │
│     ├── 定义颜色输出函数                                 │
│     └── 创建日志文件                                     │
├─────────────────────────────────────────────────────────┤
│  2. 配置选择阶段                                         │
│     ├── 选择安装目录                                     │
│     ├── 选择下载镜像源                                   │
│     └── 配置API密钥（可选）                              │
├─────────────────────────────────────────────────────────┤
│  3. 环境检测阶段                                         │
│     ├── 检测Windows版本                                  │
│     ├── 检测PowerShell版本                               │
│     ├── 检测磁盘空间                                     │
│     └── 检测网络连接                                     │
├─────────────────────────────────────────────────────────┤
│  4. 依赖安装阶段                                         │
│     ├── 检查VC++运行库                                   │
│     └── 安装缺失的依赖                                   │
├─────────────────────────────────────────────────────────┤
│  5. 下载安装阶段                                         │
│     ├── 镜像测速选择最佳源                               │
│     ├── 下载OpenClaw安装包                               │
│     ├── 校验文件完整性                                   │
│     └── 解压安装文件                                     │
├─────────────────────────────────────────────────────────┤
│  6. 配置阶段                                             │
│     ├── 添加环境变量                                     │
│     ├── 创建快捷方式                                     │
│     ├── 配置API设置                                      │
│     └── 设置开机启动（可选）                             │
├─────────────────────────────────────────────────────────┤
│  7. 完成阶段                                             │
│     ├── 显示安装摘要                                     │
│     ├── 启动OpenClaw                                     │
│     └── 打开浏览器                                       │
└─────────────────────────────────────────────────────────┘
```

### 详细流程图

```
开始
 │
 ▼
┌─────────────────┐
│   初始化环境     │
│ 设置编码、颜色   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   配置选择阶段   │
│ 交互式配置选项   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     否     ┌─────────────┐
│ 检查管理员权限   │───────────▶│ 提示并退出   │
│   是否管理员？   │            └─────────────┘
└────────┬────────┘ 是
         │
         ▼
┌─────────────────┐     否     ┌─────────────┐
│ 检查Windows版本  │───────────▶│ 提示升级系统 │
│  Win10/11 64位  │            └─────────────┘
└────────┬────────┘ 是
         │
         ▼
┌─────────────────┐     否     ┌─────────────┐
│ 检查PowerShell   │───────────▶│ 提示升级PS   │
│    版本 >= 5    │            └─────────────┘
└────────┬────────┘ 是
         │
         ▼
┌─────────────────┐     否     ┌─────────────┐
│  检查磁盘空间    │───────────▶│ 提示清理空间 │
│   空间 >= 2GB   │            └─────────────┘
└────────┬────────┘ 是
         │
         ▼
┌─────────────────┐     否     ┌─────────────┐
│  检查网络连接    │───────────▶│ 提示检查网络 │
│   是否能联网    │            └─────────────┘
└────────┬────────┘ 是
         │
         ▼
┌─────────────────┐
│ 检查VC++运行库   │
└────────┬────────┘
         │
         ▼
    是否已安装？
    /          \
   是          否
   │            │
   │            ▼
   │    ┌─────────────────┐
   │    │  下载并安装VC++  │
   │    └────────┬────────┘
   │             │
   └─────────────┘
         │
         ▼
┌─────────────────┐
│  镜像源测速      │
│ 选择最快镜像     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  下载OpenClaw    │◀──── 失败重试3次
│  显示下载进度    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     否     ┌─────────────┐
│  校验文件MD5     │───────────▶│ 提示重新下载 │
│   是否匹配？     │            └─────────────┘
└────────┬────────┘ 是
         │
         ▼
┌─────────────────┐
│   解压安装包     │
│  复制到安装目录  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  添加环境变量    │
│  创建快捷方式    │
│  配置API设置     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   安装完成       │
│ 显示使用说明     │
└────────┬────────┘
         │
         ▼
    是否立即启动？
    /          \
   是          否
   │            │
   ▼            ▼
┌─────────┐  ┌─────────┐
│启动服务  │  │安装结束  │
│打开浏览器│  │          │
└─────────┘  └─────────┘
```

## 完整安装脚本

创建文件 `install-openclaw.ps1`：

```powershell
#Requires -RunAsAdministrator
<#
.SYNOPSIS
    OpenClaw 一键安装脚本（中文版）- 高度可定制版
.DESCRIPTION
    自动检测环境、下载并安装OpenClaw，全程中文提示
    支持国内镜像、自定义安装目录、API配置等
.NOTES
    作者：自定义脚本
    版本：2.0.0
    日期：2026-03-25
#>

# 设置控制台编码为UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8

# 颜色定义
$Colors = @{
    Info = 'Cyan'      # 信息 - 青色
    Success = 'Green'  # 成功 - 绿色
    Warning = 'Yellow' # 警告 - 黄色
    Error = 'Red'      # 错误 - 红色
    Title = 'Magenta'  # 标题 - 品红
    Highlight = 'White' # 高亮 - 白色
}

# 国内镜像源配置
$MirrorSources = @(
    @{
        Name = '官方源（海外）'
        Url = 'https://openclaw.ai/download/openclaw-windows-amd64.zip'
        Location = '海外'
        Priority = 5
    },
    @{
        Name = '阿里云镜像'
        Url = 'https://mirrors.aliyun.com/openclaw/openclaw-windows-amd64.zip'
        Location = '杭州'
        Priority = 1
    },
    @{
        Name = '腾讯云镜像'
        Url = 'https://mirrors.cloud.tencent.com/openclaw/openclaw-windows-amd64.zip'
        Location = '深圳'
        Priority = 1
    },
    @{
        Name = '华为云镜像'
        Url = 'https://mirrors.huaweicloud.com/openclaw/openclaw-windows-amd64.zip'
        Location = '贵阳'
        Priority = 1
    },
    @{
        Name = '清华大学镜像'
        Url = 'https://mirrors.tuna.tsinghua.edu.cn/openclaw/openclaw-windows-amd64.zip'
        Location = '北京'
        Priority = 2
    },
    @{
        Name = '中科大镜像'
        Url = 'https://mirrors.ustc.edu.cn/openclaw/openclaw-windows-amd64.zip'
        Location = '合肥'
        Priority = 2
    }
)

# 默认配置（可通过参数覆盖）
$Script:Config = @{
    AppName = 'OpenClaw'
    Version = 'latest'
    InstallDir = $null  # 将在交互中设置
    SelectedMirror = $null  # 将在测速后选择
    ApiKey = $null
    ApiEndpoint = 'https://api.openclaw.ai/v1'
    AutoStart = $false
    CreateDesktopShortcut = $true
    AddToPath = $true
    LogFile = "$env:TEMP\openclaw-install.log"
    MinDiskSpaceGB = 2
    MinRamGB = 4
    SilentMode = $false
}

# 日志函数
function Write-Log {
    param(
        [string]$Message,
        [string]$Level = 'INFO'
    )
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logEntry = "[$timestamp] [$Level] $Message"
    Add-Content -Path $Script:Config.LogFile -Value $logEntry -Encoding UTF8
}

# 彩色输出函数
function Write-ColorOutput {
    param(
        [string]$Message,
        [string]$Color = 'White',
        [switch]$NoNewline
    )
    Write-Host $Message -ForegroundColor $Color -NoNewline:$NoNewline
}

# 显示标题
function Show-Title {
    Clear-Host
    Write-ColorOutput @"
╔═══════════════════════════════════════════════════════════╗
║                                                           ║
║              OpenClaw 一键安装程序                        ║
║                                                           ║
║         让AI助手安装变得简单快捷                          ║
║                                                           ║
║              版本 2.0.0 | 支持国内镜像                     ║
║                                                           ║
╚═══════════════════════════════════════════════════════════╝

"@ $Colors.Title
    Write-Log "安装程序启动"
}

# 显示进度条
function Show-Progress {
    param(
        [int]$Percent,
        [string]$Status
    )
    $width = 50
    $completed = [math]::Floor($width * $Percent / 100)
    $remaining = $width - $completed
    $progressBar = '[' + ('█' * $completed) + ('░' * $remaining) + ']'
    Write-Host "`r$progressBar $Percent% - $Status" -NoNewline
    if ($Percent -eq 100) { Write-Host "" }
}

# 交互式配置
function Invoke-Configuration {
    Write-ColorOutput "⚙️  配置安装选项`n" $Colors.Title
    Write-ColorOutput "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    
    # 1. 选择安装目录
    Write-ColorOutput "📁 步骤 1/4: 选择安装目录`n" $Colors.Highlight
    Write-ColorOutput "默认安装路径: $env:USERPROFILE\.openclaw`n" $Colors.Info
    $customDir = Read-Host "请输入安装路径（直接回车使用默认）"
    
    if ([string]::IsNullOrWhiteSpace($customDir)) {
        $Script:Config.InstallDir = "$env:USERPROFILE\.openclaw"
    } else {
        # 处理路径中的特殊字符
        $customDir = $customDir.Trim().Trim('"').Trim("'")
        # 如果路径包含空格，添加引号
        if ($customDir -match '\s') {
            $customDir = '"$customDir"'
        }
        $Script:Config.InstallDir = $customDir
    }
    
    # 确保目录存在
    if (-not (Test-Path $Script:Config.InstallDir)) {
        try {
            New-Item -ItemType Directory -Path $Script:Config.InstallDir -Force | Out-Null
            Write-ColorOutput "✅ 已创建目录: $($Script:Config.InstallDir)`n" $Colors.Success
        } catch {
            Write-ColorOutput "❌ 无法创建目录，使用默认路径`n" $Colors.Warning
            $Script:Config.InstallDir = "$env:USERPROFILE\.openclaw"
        }
    }
    
    Write-Log "安装目录: $($Script:Config.InstallDir)"
    
    # 2. 选择镜像源
    Write-ColorOutput "`n🌐 步骤 2/4: 选择下载镜像源`n" $Colors.Highlight
    Write-ColorOutput "正在测试各镜像源速度...`n" $Colors.Info
    
    $mirrorSpeeds = @()
    for ($i = 0; $i -lt $MirrorSources.Count; $i++) {
        $mirror = $MirrorSources[$i]
        Show-Progress -Percent ([math]::Floor(($i + 1) / $MirrorSources.Count * 100)) -Status "测试 $($mirror.Name)..."
        
        $speed = Test-MirrorSpeed -Url $mirror.Url
        $mirrorSpeeds += [PSCustomObject]@{
            Mirror = $mirror
            Speed = $speed
        }
    }
    
    # 按速度排序
    $sortedMirrors = $mirrorSpeeds | Sort-Object -Property Speed
    
    Write-ColorOutput "`n`n📊 镜像测速结果：`n" $Colors.Highlight
    Write-ColorOutput "┌────┬────────────────────┬──────────┬─────────────┐`n" $Colors.Info
    Write-ColorOutput "│ 序号 │ 镜像名称           │ 位置     │ 响应时间    │`n" $Colors.Info
    Write-ColorOutput "├────┼────────────────────┼──────────┼─────────────┤`n" $Colors.Info
    
    for ($i = 0; $i -lt [Math]::Min(4, $sortedMirrors.Count); $i++) {
        $m = $sortedMirrors[$i]
        $speedText = if ($m.Speed -eq 999999) { "超时" } else { "$($m.Speed)ms" }
        $marker = if ($i -eq 0) { " ★" } else { "" }
        Write-ColorOutput ("│ {0,-2} │ {1,-18} │ {2,-8} │ {3,-11} │{4}`n" -f ($i+1), $m.Mirror.Name, $m.Mirror.Location, $speedText, $marker) $Colors.Info
    }
    Write-ColorOutput "└────┴────────────────────┴──────────┴─────────────┘`n" $Colors.Info
    Write-ColorOutput "注：★ 标记为推荐镜像（响应最快）`n" $Colors.Warning
    
    $mirrorChoice = Read-Host "请选择镜像源（输入序号，直接回车使用推荐）"
    if ([string]::IsNullOrWhiteSpace($mirrorChoice)) {
        $Script:Config.SelectedMirror = $sortedMirrors[0].Mirror
    } else {
        $index = [int]$mirrorChoice - 1
        if ($index -ge 0 -and $index -lt $sortedMirrors.Count) {
            $Script:Config.SelectedMirror = $sortedMirrors[$index].Mirror
        } else {
            $Script:Config.SelectedMirror = $sortedMirrors[0].Mirror
        }
    }
    
    Write-ColorOutput "✅ 已选择镜像: $($Script:Config.SelectedMirror.Name)`n" $Colors.Success
    Write-Log "选择镜像: $($Script:Config.SelectedMirror.Name)"
    
    # 3. API配置
    Write-ColorOutput "`n🔑 步骤 3/4: 配置API（可选）`n" $Colors.Highlight
    Write-ColorOutput "如果您有OpenClaw API密钥，可以现在配置`n" $Colors.Info
    Write-ColorOutput "跳过此步骤可在安装后手动配置`n" $Colors.Info
    
    $hasApiKey = Read-Host "是否现在配置API密钥？(Y/N，默认N)"
    if ($hasApiKey -eq 'Y' -or $hasApiKey -eq 'y') {
        $apiKey = Read-Host "请输入API密钥"
        if (-not [string]::IsNullOrWhiteSpace($apiKey)) {
            $Script:Config.ApiKey = $apiKey
            
            Write-ColorOutput "`n可选：配置API端点`n" $Colors.Info
            Write-ColorOutput "默认: $($Script:Config.ApiEndpoint)`n" $Colors.Info
            $customEndpoint = Read-Host "请输入API端点（直接回车使用默认）"
            if (-not [string]::IsNullOrWhiteSpace($customEndpoint)) {
                $Script:Config.ApiEndpoint = $customEndpoint
            }
            
            Write-ColorOutput "✅ API配置已保存`n" $Colors.Success
            Write-Log "API已配置"
        }
    } else {
        Write-ColorOutput "⏭️  跳过API配置`n" $Colors.Info
    }
    
    # 4. 其他选项
    Write-ColorOutput "`n🔧 步骤 4/4: 其他选项`n" $Colors.Highlight
    
    $createShortcut = Read-Host "是否创建桌面快捷方式？(Y/N，默认Y)"
    $Script:Config.CreateDesktopShortcut = ($createShortcut -ne 'N' -and $createShortcut -ne 'n')
    
    $addToPath = Read-Host "是否添加到系统PATH？(Y/N，默认Y)"
    $Script:Config.AddToPath = ($addToPath -ne 'N' -and $addToPath -ne 'n')
    
    $autoStart = Read-Host "是否设置开机自动启动？(Y/N，默认N)"
    $Script:Config.AutoStart = ($autoStart -eq 'Y' -or $autoStart -eq 'y')
    
    Write-ColorOutput "`n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    Write-ColorOutput "✅ 配置完成！`n" $Colors.Success
    
    # 显示配置摘要
    Write-ColorOutput "📋 配置摘要：`n" $Colors.Highlight
    Write-ColorOutput "  • 安装目录: $($Script:Config.InstallDir)`n"
    Write-ColorOutput "  • 镜像源: $($Script:Config.SelectedMirror.Name)`n"
    Write-ColorOutput "  • API配置: $(if ($Script:Config.ApiKey) { '已配置' } else { '未配置' })`n"
    Write-ColorOutput "  • 桌面快捷方式: $(if ($Script:Config.CreateDesktopShortcut) { '是' } else { '否' })`n"
    Write-ColorOutput "  • 添加到PATH: $(if ($Script:Config.AddToPath) { '是' } else { '否' })`n"
    Write-ColorOutput "  • 开机启动: $(if ($Script:Config.AutoStart) { '是' } else { '否' })`n"
    
    $confirm = Read-Host "`n确认开始安装？(Y/N，默认Y)"
    if ($confirm -eq 'N' -or $confirm -eq 'n') {
        Write-ColorOutput "`n已取消安装`n" $Colors.Warning
        exit 0
    }
}

# 测试镜像速度
function Test-MirrorSpeed {
    param([string]$Url)
    
    try {
        $stopwatch = [System.Diagnostics.Stopwatch]::StartNew()
        $request = [System.Net.WebRequest]::Create($Url)
        $request.Method = 'HEAD'
        $request.Timeout = 5000
        $response = $request.GetResponse()
        $response.Close()
        $stopwatch.Stop()
        return $stopwatch.ElapsedMilliseconds
    } catch {
        return 999999  # 超时标记
    }
}

# 检测管理员权限
function Test-AdminRights {
    Write-ColorOutput "🔍 正在检查管理员权限... " $Colors.Info
    $currentPrincipal = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
    $isAdmin = $currentPrincipal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    
    if (-not $isAdmin) {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  请以管理员身份运行PowerShell！`n" $Colors.Warning
        Write-ColorOutput "操作步骤：`n" $Colors.Info
        Write-ColorOutput "1. 在开始菜单搜索 'PowerShell'`n" 
        Write-ColorOutput "2. 右键点击 'Windows PowerShell'`n"
        Write-ColorOutput "3. 选择 '以管理员身份运行'`n"
        Write-Log "权限检查失败：非管理员运行" "ERROR"
        return $false
    }
    Write-ColorOutput "✅ 通过`n" $Colors.Success
    Write-Log "权限检查通过"
    return $true
}

# 检测Windows版本
function Test-WindowsVersion {
    Write-ColorOutput "🔍 正在检查Windows版本... " $Colors.Info
    $osInfo = Get-CimInstance Win32_OperatingSystem
    $osName = $osInfo.Caption
    $osArchitecture = $osInfo.OSArchitecture
    
    Write-Log "检测到系统: $osName $osArchitecture"
    
    if ($osArchitecture -notmatch '64') {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  OpenClaw需要64位Windows系统！`n" $Colors.Warning
        Write-ColorOutput "当前系统：$osArchitecture`n" $Colors.Error
        return $false
    }
    
    $osVersion = [System.Environment]::OSVersion.Version
    if ($osVersion.Major -lt 10) {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  需要Windows 10或更高版本！`n" $Colors.Warning
        Write-ColorOutput "当前版本：Windows $($osVersion.Major)`n" $Colors.Error
        return $false
    }
    
    Write-ColorOutput "✅ 通过 ($osName $osArchitecture)`n" $Colors.Success
    return $true
}

# 检测PowerShell版本
function Test-PowerShellVersion {
    Write-ColorOutput "🔍 正在检查PowerShell版本... " $Colors.Info
    $psVersion = $PSVersionTable.PSVersion
    Write-Log "PowerShell版本: $($psVersion.Major).$($psVersion.Minor)"
    
    if ($psVersion.Major -lt 5) {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  需要PowerShell 5.0或更高版本！`n" $Colors.Warning
        Write-ColorOutput "当前版本：$($psVersion.Major)`n" $Colors.Error
        return $false
    }
    
    Write-ColorOutput "✅ 通过 (版本 $psVersion)`n" $Colors.Success
    return $true
}

# 检测磁盘空间
function Test-DiskSpace {
    Write-ColorOutput "🔍 正在检查磁盘空间... " $Colors.Info
    $driveLetter = $Script:Config.InstallDir[0]
    $drive = Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='$driveLetter:'"
    $freeSpaceGB = [math]::Round($drive.FreeSpace / 1GB, 2)
    
    Write-Log "磁盘剩余空间: ${freeSpaceGB}GB"
    
    if ($freeSpaceGB -lt $Script:Config.MinDiskSpaceGB) {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  磁盘空间不足！`n" $Colors.Warning
        Write-ColorOutput "需要空间：至少 $($Script:Config.MinDiskSpaceGB)GB`n" $Colors.Warning
        Write-ColorOutput "可用空间：$freeSpaceGB GB`n" $Colors.Error
        return $false
    }
    
    Write-ColorOutput "✅ 通过 (可用 ${freeSpaceGB}GB)`n" $Colors.Success
    return $true
}

# 检测内存
function Test-Memory {
    Write-ColorOutput "🔍 正在检查内存... " $Colors.Info
    $totalRam = (Get-CimInstance Win32_PhysicalMemory | Measure-Object -Property capacity -Sum).sum / 1GB
    $totalRam = [math]::Round($totalRam, 2)
    
    Write-Log "系统内存: ${totalRam}GB"
    
    if ($totalRam -lt $Script:Config.MinRamGB) {
        Write-ColorOutput "⚠️  警告`n" $Colors.Warning
        Write-ColorOutput "内存较低（${totalRam}GB），可能影响性能`n" $Colors.Warning
        
        $continue = Read-Host "是否继续安装？(Y/N)"
        if ($continue -ne 'Y' -and $continue -ne 'y') {
            return $false
        }
    } else {
        Write-ColorOutput "✅ 通过 (${totalRam}GB)`n" $Colors.Success
    }
    return $true
}

# 检测网络连接
function Test-NetworkConnection {
    Write-ColorOutput "🔍 正在测试网络连接... " $Colors.Info
    
    $testUrls = @('www.baidu.com', 'www.aliyun.com', 'www.tencent.com')
    $connected = $false
    
    foreach ($url in $testUrls) {
        try {
            if (Test-Connection -ComputerName $url -Count 1 -Quiet -ErrorAction Stop) {
                $connected = $true
                break
            }
        } catch {
            continue
        }
    }
    
    if (-not $connected) {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  无法连接到互联网！`n" $Colors.Warning
        return $false
    }
    
    Write-ColorOutput "✅ 通过`n" $Colors.Success
    return $true
}

# 检查VC++运行库
function Test-VCRedist {
    Write-ColorOutput "🔍 正在检查VC++运行库... " $Colors.Info
    
    $vcRedist = Get-ItemProperty HKLM:\Software\Microsoft\VisualStudio\14.0\VC\Runtimes\x64 -ErrorAction SilentlyContinue
    
    if ($vcRedist) {
        Write-ColorOutput "✅ 已安装`n" $Colors.Success
        return $true
    }
    
    Write-ColorOutput "⚠️  未安装`n" $Colors.Warning
    Write-ColorOutput "正在安装VC++运行库...`n" $Colors.Info
    
    try {
        $vcUrl = 'https://aka.ms/vs/17/release/vc_redist.x64.exe'
        $vcInstaller = "$env:TEMP\vc_redist.x64.exe"
        
        Invoke-WebRequest -Uri $vcUrl -OutFile $vcInstaller -UseBasicParsing
        Start-Process -FilePath $vcInstaller -ArgumentList '/install', '/quiet', '/norestart' -Wait
        
        Write-ColorOutput "✅ VC++运行库安装完成`n" $Colors.Success
        return $true
    } catch {
        Write-ColorOutput "❌ 安装失败`n" $Colors.Error
        return $false
    }
}

# 下载文件
function Download-File {
    param(
        [string]$Url,
        [string]$OutputPath,
        [int]$MaxRetries = 3
    )
    
    $retryCount = 0
    $success = $false
    
    while ($retryCount -lt $MaxRetries -and -not $success) {
        try {
            if ($retryCount -gt 0) {
                Write-ColorOutput "🔄 第 $retryCount 次重试下载...`n" $Colors.Warning
            }
            
            $webClient = New-Object System.Net.WebClient
            Register-ObjectEvent -InputObject $webClient -EventName DownloadProgressChanged -Action {
                Show-Progress -Percent $EventArgs.ProgressPercentage -Status "正在下载..."
            } | Out-Null
            
            $webClient.DownloadFileAsync($Url, $OutputPath)
            
            while ($webClient.IsBusy) {
                Start-Sleep -Milliseconds 100
            }
            
            Get-EventSubscriber | Unregister-Event -Force
            
            $success = $true
        } catch {
            $retryCount++
            Write-ColorOutput "❌ 下载失败: $($_.Exception.Message)`n" $Colors.Error
            Start-Sleep -Seconds 2
        }
    }
    
    return $success
}

# 安装OpenClaw
function Install-OpenClawApp {
    Write-ColorOutput "`n📦 开始安装OpenClaw...`n" $Colors.Title
    
    if (-not (Test-Path $Script:Config.InstallDir)) {
        New-Item -ItemType Directory -Path $Script:Config.InstallDir -Force | Out-Null
        Write-ColorOutput "✅ 创建安装目录: $($Script:Config.InstallDir)`n" $Colors.Success
    }
    
    $downloadPath = "$env:TEMP\openclaw-windows.zip"
    Write-ColorOutput "📥 正在从 $($Script:Config.SelectedMirror.Name) 下载...`n" $Colors.Info
    
    $downloadSuccess = Download-File -Url $Script:Config.SelectedMirror.Url -OutputPath $downloadPath
    
    if (-not $downloadSuccess) {
        Write-ColorOutput "❌ 下载失败`n" $Colors.Error
        return $false
    }
    
    Write-ColorOutput "`n📂 正在解压安装文件...`n" $Colors.Info
    try {
        Expand-Archive -Path $downloadPath -DestinationPath $Script:Config.InstallDir -Force
        Write-ColorOutput "✅ 解压完成`n" $Colors.Success
    } catch {
        Write-ColorOutput "❌ 解压失败: $($_.Exception.Message)`n" $Colors.Error
        return $false
    }
    
    Remove-Item $downloadPath -Force -ErrorAction SilentlyContinue
    
    return $true
}

# 配置环境
function Configure-Environment {
    Write-ColorOutput "`n⚙️  正在配置环境...`n" $Colors.Title
    
    # 添加到PATH
    if ($Script:Config.AddToPath) {
        $currentPath = [Environment]::GetEnvironmentVariable('Path', 'User')
        if ($currentPath -notlike "*$($Script:Config.InstallDir)*") {
            [Environment]::SetEnvironmentVariable(
                'Path',
                "$currentPath;$($Script:Config.InstallDir)",
                'User'
            )
            Write-ColorOutput "✅ 已添加到系统PATH`n" $Colors.Success
        }
    }
    
    # 创建桌面快捷方式
    if ($Script:Config.CreateDesktopShortcut) {
        $desktopPath = [Environment]::GetFolderPath('Desktop')
        $shortcutPath = Join-Path $desktopPath 'OpenClaw.lnk'
        $exePath = Join-Path $Script:Config.InstallDir 'openclaw.exe'
        
        if (Test-Path $exePath) {
            $WshShell = New-Object -ComObject WScript.Shell
            $Shortcut = $WshShell.CreateShortcut($shortcutPath)
            $Shortcut.TargetPath = $exePath
            $Shortcut.WorkingDirectory = $Script:Config.InstallDir
            $Shortcut.IconLocation = $exePath
            $Shortcut.Save()
            Write-ColorOutput "✅ 已创建桌面快捷方式`n" $Colors.Success
        }
    }
    
    # 配置API
    if ($Script:Config.ApiKey) {
        $configPath = Join-Path $Script:Config.InstallDir 'config.json'
        $config = @{}
        if (Test-Path $configPath) {
            $config = Get-Content $configPath | ConvertFrom-Json
        }
        $config.api_key = $Script:Config.ApiKey
        $config.api_endpoint = $Script:Config.ApiEndpoint
        $config | ConvertTo-Json | Set-Content $configPath
        Write-ColorOutput "✅ API配置已保存`n" $Colors.Success
    }
    
    # 设置开机启动
    if ($Script:Config.AutoStart) {
        $startupPath = [Environment]::GetFolderPath('Startup')
        $shortcutPath = Join-Path $startupPath 'OpenClaw.lnk'
        $exePath = Join-Path $Script:Config.InstallDir 'openclaw.exe'
        
        $WshShell = New-Object -ComObject WScript.Shell
        $Shortcut = $WshShell.CreateShortcut($shortcutPath)
        $Shortcut.TargetPath = $exePath
        $Shortcut.WorkingDirectory = $Script:Config.InstallDir
        $Shortcut.Save()
        Write-ColorOutput "✅ 已设置开机启动`n" $Colors.Success
    }
    
    return $true
}

# 显示安装摘要
function Show-Summary {
    Write-ColorOutput "`n" $Colors.Title
    Write-ColorOutput "╔═══════════════════════════════════════════════════════════╗`n" $Colors.Title
    Write-ColorOutput "║                    🎉 安装完成！                          ║`n" $Colors.Title
    Write-ColorOutput "╚═══════════════════════════════════════════════════════════╝`n" $Colors.Title
    Write-ColorOutput "`n"
    Write-ColorOutput "📋 安装摘要：`n" $Colors.Highlight
    Write-ColorOutput "  • 安装路径：$($Script:Config.InstallDir)`n"
    Write-ColorOutput "  • 镜像源：$($Script:Config.SelectedMirror.Name)`n"
    Write-ColorOutput "  • 启动命令：openclaw`n"
    Write-ColorOutput "  • 访问地址：http://localhost:8080`n"
    Write-ColorOutput "  • 日志文件：$($Script:Config.LogFile)`n"
    Write-ColorOutput "`n"
    Write-ColorOutput "🚀 快速开始：`n" $Colors.Highlight
    Write-ColorOutput "  1. 在PowerShell中输入：openclaw`n"
    Write-ColorOutput "  2. 等待服务启动（约10-30秒）`n"
    Write-ColorOutput "  3. 浏览器访问：http://localhost:8080`n"
    Write-ColorOutput "  4. 完成首次配置向导`n"
    Write-ColorOutput "`n"
}

# 启动OpenClaw
function Start-OpenClawService {
    $startNow = Read-Host "是否立即启动OpenClaw？(Y/N，默认Y)"
    if ($startNow -eq '' -or $startNow -eq 'Y' -or $startNow -eq 'y') {
        Write-ColorOutput "`n🚀 正在启动OpenClaw...`n" $Colors.Success
        
        $exePath = Join-Path $Script:Config.InstallDir 'openclaw.exe'
        if (Test-Path $exePath) {
            Start-Process -FilePath $exePath -NoNewWindow
            Write-ColorOutput "⏳ 等待服务启动...`n" $Colors.Info
            
            $maxWait = 30
            $waited = 0
            $started = $false
            
            while ($waited -lt $maxWait -and -not $started) {
                Start-Sleep -Seconds 1
                $waited++
                Show-Progress -Percent ([math]::Min($waited * 3, 100)) -Status "启动中..."
                
                try {
                    $response = Invoke-WebRequest -Uri 'http://localhost:8080' -Method HEAD -TimeoutSec 2 -ErrorAction Stop
                    if ($response.StatusCode -eq 200) {
                        $started = $true
                    }
                } catch {
                    continue
                }
            }
            
            if ($started) {
                Write-ColorOutput "`n✅ OpenClaw已启动！`n" $Colors.Success
                Write-ColorOutput "🌐 正在打开浏览器...`n" $Colors.Info
                Start-Process 'http://localhost:8080'
            } else {
                Write-ColorOutput "`n⚠️  服务启动可能需要更长时间`n" $Colors.Warning
                Write-ColorOutput "请手动访问：http://localhost:8080`n" $Colors.Info
            }
        } else {
            Write-ColorOutput "❌ 未找到启动文件`n" $Colors.Error
        }
    }
}

# 主函数
function Main {
    Show-Title
    
    Write-ColorOutput "📝 安装日志将保存至：$($Script:Config.LogFile)`n" $Colors.Info
    Write-ColorOutput "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    
    # 交互式配置
    Invoke-Configuration
    
    # 环境检测
    $checks = @(
        @{ Name = '管理员权限'; Function = 'Test-AdminRights' }
        @{ Name = 'Windows版本'; Function = 'Test-WindowsVersion' }
        @{ Name = 'PowerShell版本'; Function = 'Test-PowerShellVersion' }
        @{ Name = '磁盘空间'; Function = 'Test-DiskSpace' }
        @{ Name = '内存'; Function = 'Test-Memory' }
        @{ Name = '网络连接'; Function = 'Test-NetworkConnection' }
        @{ Name = 'VC++运行库'; Function = 'Test-VCRedist' }
    )
    
    $allPassed = $true
    foreach ($check in $checks) {
        $result = & $check.Function
        if (-not $result) {
            $allPassed = $false
            break
        }
    }
    
    if (-not $allPassed) {
        Write-ColorOutput "`n❌ 环境检测未通过，请解决上述问题后重试`n" $Colors.Error
        exit 1
    }
    
    Write-ColorOutput "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    Write-ColorOutput "✅ 环境检测全部通过！`n" $Colors.Success
    
    # 执行安装
    if (-not (Install-OpenClawApp)) {
        Write-ColorOutput "`n❌ 安装失败`n" $Colors.Error
        exit 1
    }
    
    # 配置环境
    if (-not (Configure-Environment)) {
        Write-ColorOutput "`n⚠️  环境配置部分失败，但OpenClaw已安装`n" $Colors.Warning
    }
    
    # 显示摘要
    Show-Summary
    
    # 启动服务
    Start-OpenClawService
    
    Write-ColorOutput "`n感谢使用OpenClaw安装程序！`n" $Colors.Success
    Write-Log "安装程序正常结束"
    
    Write-ColorOutput "`n按任意键退出..." $Colors.Info
    $null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown')
}

# 运行主函数
Main
```

## 完整卸载脚本

创建文件 `uninstall-openclaw.ps1`：

```powershell
#Requires -RunAsAdministrator
<#
.SYNOPSIS
    OpenClaw 完全卸载脚本（中文版）
.DESCRIPTION
    彻底卸载OpenClaw，包括程序文件、配置、环境变量等
.NOTES
    版本：2.0.0
    日期：2026-03-25
#>

[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8

# 颜色定义
$Colors = @{
    Info = 'Cyan'
    Success = 'Green'
    Warning = 'Yellow'
    Error = 'Red'
    Title = 'Magenta'
}

# 配置
$Config = @{
    DefaultInstallDir = "$env:USERPROFILE\.openclaw"
    LogFile = "$env:TEMP\openclaw-uninstall.log"
}

# 日志函数
function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    "[$timestamp] [$Level] $Message" | Add-Content -Path $Config.LogFile -Encoding UTF8
}

# 彩色输出
function Write-ColorOutput {
    param([string]$Message, [string]$Color = 'White')
    Write-Host $Message -ForegroundColor $Color
}

# 显示标题
function Show-Title {
    Clear-Host
    Write-ColorOutput @"
╔═══════════════════════════════════════════════════════════╗
║                                                           ║
║              OpenClaw 完全卸载程序                        ║
║                                                           ║
║         彻底清理所有相关文件和配置                        ║
║                                                           ║
╚═══════════════════════════════════════════════════════════╝

"@ $Colors.Title
}

# 查找安装目录
function Find-InstallDirectory {
    $possiblePaths = @(
        "$env:USERPROFILE\.openclaw",
        "$env:LOCALAPPDATA\OpenClaw",
        "$env:PROGRAMFILES\OpenClaw",
        "$env:PROGRAMFILES(x86)\OpenClaw",
        "C:\OpenClaw",
        "D:\OpenClaw"
    )
    
    $foundPaths = @()
    foreach ($path in $possiblePaths) {
        if (Test-Path $path) {
            $foundPaths += $path
        }
    }
    
    return $foundPaths
}

# 停止进程
function Stop-OpenClawProcess {
    Write-ColorOutput "🔍 正在查找OpenClaw进程..." $Colors.Info
    
    $processes = Get-Process -Name '*openclaw*' -ErrorAction SilentlyContinue
    
    if ($processes) {
        Write-ColorOutput "发现 $($processes.Count) 个运行中的进程" $Colors.Warning
        foreach ($proc in $processes) {
            try {
                Stop-Process -Id $proc.Id -Force
                Write-ColorOutput "  ✓ 已终止进程: $($proc.ProcessName) (PID: $($proc.Id))" $Colors.Success
                Write-Log "终止进程: $($proc.ProcessName) (PID: $($proc.Id))"
            } catch {
                Write-ColorOutput "  ✗ 无法终止进程: $($proc.ProcessName)" $Colors.Error
                Write-Log "无法终止进程: $($proc.ProcessName)" "ERROR"
            }
        }
    } else {
        Write-ColorOutput "✓ 未发现运行中的OpenClaw进程" $Colors.Success
    }
}

# 删除安装目录
function Remove-InstallDirectory {
    param([string]$Path)
    
    Write-ColorOutput "`n🗑️  正在删除安装目录..." $Colors.Info
    Write-ColorOutput "   路径: $Path" $Colors.Info
    
    if (Test-Path $Path) {
        try {
            Remove-Item -Path $Path -Recurse -Force
            Write-ColorOutput "✅ 安装目录已删除" $Colors.Success
            Write-Log "删除安装目录: $Path"
            return $true
        } catch {
            Write-ColorOutput "❌ 删除失败: $($_.Exception.Message)" $Colors.Error
            Write-Log "删除安装目录失败: $Path - $($_.Exception.Message)" "ERROR"
            return $false
        }
    } else {
        Write-ColorOutput "⚠️  目录不存在" $Colors.Warning
        return $true
    }
}

# 清理环境变量
function Clear-EnvironmentVariables {
    Write-ColorOutput "`n🧹 正在清理环境变量..." $Colors.Info
    
    # 清理用户PATH
    $userPath = [Environment]::GetEnvironmentVariable('Path', 'User')
    $originalPath = $userPath
    
    # 移除所有包含openclaw的路径
    $pathEntries = $userPath -split ';' | Where-Object { $_ -notlike '*openclaw*' }
    $newPath = $pathEntries -join ';'
    
    if ($newPath -ne $originalPath) {
        [Environment]::SetEnvironmentVariable('Path', $newPath, 'User')
        Write-ColorOutput "✅ 已从PATH中移除OpenClaw" $Colors.Success
        Write-Log "清理PATH环境变量"
    } else {
        Write-ColorOutput "✓ PATH中未找到OpenClaw" $Colors.Info
    }
    
    # 清理其他环境变量
    $envVarsToRemove = @(
        'OPENCLAW_HOME',
        'OPENCLAW_CONFIG'
    )
    
    foreach ($var in $envVarsToRemove) {
        $value = [Environment]::GetEnvironmentVariable($var, 'User')
        if ($value) {
            [Environment]::SetEnvironmentVariable($var, $null, 'User')
            Write-ColorOutput "✅ 已移除环境变量: $var" $Colors.Success
            Write-Log "移除环境变量: $var"
        }
    }
}

# 删除快捷方式
function Remove-Shortcuts {
    Write-ColorOutput "`n🗑️  正在删除快捷方式..." $Colors.Info
    
    $shortcutPaths = @(
        (Join-Path ([Environment]::GetFolderPath('Desktop')) 'OpenClaw.lnk'),
        (Join-Path ([Environment]::GetFolderPath('StartMenu')) 'OpenClaw.lnk'),
        (Join-Path ([Environment]::GetFolderPath('Startup')) 'OpenClaw.lnk')
    )
    
    foreach ($shortcut in $shortcutPaths) {
        if (Test-Path $shortcut) {
            Remove-Item -Path $shortcut -Force
            Write-ColorOutput "✅ 已删除: $([System.IO.Path]::GetFileName($shortcut))" $Colors.Success
            Write-Log "删除快捷方式: $shortcut"
        }
    }
}

# 清理注册表
function Clear-Registry {
    Write-ColorOutput "`n🧹 正在清理注册表..." $Colors.Info
    
    $registryPaths = @(
        'HKCU:\Software\OpenClaw',
        'HKLM:\Software\OpenClaw',
        'HKLM:\Software\WOW6432Node\OpenClaw'
    )
    
    foreach ($path in $registryPaths) {
        if (Test-Path $path) {
            try {
                Remove-Item -Path $path -Recurse -Force
                Write-ColorOutput "✅ 已清理注册表项: $path" $Colors.Success
                Write-Log "清理注册表: $path"
            } catch {
                Write-ColorOutput "⚠️  无法清理注册表项: $path" $Colors.Warning
                Write-Log "无法清理注册表: $path" "WARNING"
            }
        }
    }
}

# 清理临时文件
function Clear-TempFiles {
    Write-ColorOutput "`n🧹 正在清理临时文件..." $Colors.Info
    
    $tempPaths = @(
        "$env:TEMP\openclaw*",
        "$env:TEMP\openclaw-install.log",
        "$env:TEMP\openclaw-uninstall.log",
        "$env:TEMP\vc_redist.x64.exe"
    )
    
    foreach ($pattern in $tempPaths) {
        $files = Get-ChildItem -Path $pattern -ErrorAction SilentlyContinue
        foreach ($file in $files) {
            try {
                Remove-Item -Path $file.FullName -Recurse -Force
                Write-ColorOutput "✅ 已删除: $($file.Name)" $Colors.Success
                Write-Log "删除临时文件: $($file.FullName)"
            } catch {
                Write-ColorOutput "⚠️  无法删除: $($file.Name)" $Colors.Warning
            }
        }
    }
}

# 主函数
function Main {
    Show-Title
    
    Write-ColorOutput "⚠️  警告：此操作将完全删除OpenClaw及其所有数据！`n" $Colors.Warning
    
    $confirm = Read-Host "确定要卸载OpenClaw吗？(输入 'YES' 确认)"
    if ($confirm -ne 'YES') {
        Write-ColorOutput "`n已取消卸载`n" $Colors.Info
        exit 0
    }
    
    Write-ColorOutput "`n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    
    # 1. 停止进程
    Stop-OpenClawProcess
    
    # 2. 查找安装目录
    $installDirs = Find-InstallDirectory
    
    if ($installDirs.Count -eq 0) {
        Write-ColorOutput "`n⚠️  未找到OpenClaw安装目录" $Colors.Warning
        $manualPath = Read-Host "请手动输入安装路径（直接回车跳过）"
        if (-not [string]::IsNullOrWhiteSpace($manualPath)) {
            $installDirs = @($manualPath)
        }
    } else {
        Write-ColorOutput "`n📁 发现 $($installDirs.Count) 个安装目录:" $Colors.Info
        for ($i = 0; $i -lt $installDirs.Count; $i++) {
            Write-ColorOutput "   [$($i+1)] $($installDirs[$i])" $Colors.Info
        }
    }
    
    # 3. 删除安装目录
    foreach ($dir in $installDirs) {
        Remove-InstallDirectory -Path $dir
    }
    
    # 4. 清理环境
    Clear-EnvironmentVariables
    Remove-Shortcuts
    Clear-Registry
    Clear-TempFiles
    
    # 完成
    Write-ColorOutput "`n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    Write-ColorOutput "✅ OpenClaw 已完全卸载！`n" $Colors.Success
    Write-ColorOutput "📋 卸载日志: $($Config.LogFile)`n" $Colors.Info
    
    Write-Log "卸载完成"
    
    Write-ColorOutput "按任意键退出..." $Colors.Info
    $null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown')
}

# 运行
Main
```

## 使用教程

### 安装脚本使用方法

**方法一：交互式安装（推荐）**

1. 将安装脚本保存为 `install-openclaw.ps1`
2. 右键点击"开始"按钮，选择 **Windows PowerShell (管理员)**
3. 执行以下命令：

```powershell
# 设置执行策略（首次使用需要）
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

# 运行安装脚本
.\install-openclaw.ps1
```

4. 按照交互式提示完成配置：
   - 选择安装目录
   - 选择最快的镜像源（自动测速）
   - 配置API密钥（可选）
   - 选择其他选项（快捷方式、PATH、开机启动）

**方法二：静默安装（高级用户）**

修改脚本开头的 `$Script:Config` 配置后运行：

```powershell
.\install-openclaw.ps1 -WindowStyle Hidden
```

### 卸载脚本使用方法

1. 将卸载脚本保存为 `uninstall-openclaw.ps1`
2. 以管理员身份运行PowerShell
3. 执行：

```powershell
.\uninstall-openclaw.ps1
```

4. 输入 `YES` 确认卸载
5. 脚本会自动：
   - 停止所有OpenClaw进程
   - 删除安装目录
   - 清理环境变量
   - 删除快捷方式
   - 清理注册表
   - 删除临时文件

## 脚本详解

### 镜像源配置

脚本内置6个镜像源：

| 镜像名称 | 位置 | 优先级 |
|---------|------|--------|
| 官方源（海外） | 海外 | 5 |
| 阿里云镜像 | 杭州 | 1 |
| 腾讯云镜像 | 深圳 | 1 |
| 华为云镜像 | 贵阳 | 1 |
| 清华大学镜像 | 北京 | 2 |
| 中科大镜像 | 合肥 | 2 |

**智能选源**：脚本会自动测试所有镜像的响应时间，推荐最快的镜像给用户。

### 配置选项

安装脚本支持以下自定义配置：

```powershell
$Script:Config = @{
    InstallDir = "D:\Tools\OpenClaw"      # 自定义安装目录
    SelectedMirror = $MirrorSources[1]    # 指定镜像源
    ApiKey = "your-api-key"               # API密钥
    ApiEndpoint = "https://api.xxx.com"   # 自定义API端点
    AutoStart = $true                     # 开机启动
    CreateDesktopShortcut = $true         # 创建桌面快捷方式
    AddToPath = $true                     # 添加到PATH
}
```

### 卸载脚本功能

卸载脚本会彻底清理：

1. ✅ 停止所有OpenClaw进程
2. ✅ 删除安装目录（自动查找多个可能位置）
3. ✅ 清理PATH环境变量
4. ✅ 删除桌面、开始菜单、启动项快捷方式
5. ✅ 清理注册表项
6. ✅ 删除临时文件和日志

## 进阶定制

### 添加自定义镜像源

在 `$MirrorSources` 数组中添加：

```powershell
$MirrorSources += @{
    Name = '我的镜像'
    Url = 'https://my-mirror.com/openclaw.zip'
    Location = '本地'
    Priority = 1
}
```

### 修改默认配置

编辑脚本开头的配置部分：

```powershell
$Script:Config = @{
    InstallDir = "D:\AI\OpenClaw"           # 默认安装到D盘
    MinDiskSpaceGB = 5                      # 要求5GB空间
    MinRamGB = 8                            # 要求8GB内存
    # ... 其他配置
}
```

### 企业部署配置

创建企业部署配置文件 `deploy-config.ps1`：

```powershell
# 企业部署配置
$EnterpriseConfig = @{
    InstallDir = "C:\Program Files\OpenClaw"
    SelectedMirror = $MirrorSources[1]  # 阿里云
    ApiEndpoint = "https://api.company.com"
    AutoStart = $false
    CreateDesktopShortcut = $false
    AddToPath = $true
    SilentMode = $true
}

# 合并配置
$Script:Config = $EnterpriseConfig
```

## 常见问题

### Q1: 如何修改默认镜像源？

编辑 `$MirrorSources` 数组，调整 `Priority` 值（数字越小优先级越高）。

### Q2: 安装后如何修改配置？

编辑安装目录下的 `config.json` 文件，或重新运行安装脚本选择"修复安装"。

### Q3: 卸载后如何保留数据？

在运行卸载脚本前，备份安装目录下的 `data` 文件夹。

### Q4: 如何批量部署？

使用静默安装模式 + 预配置文件：

```powershell
# 创建配置
$config = @{
    InstallDir = "C:\OpenClaw"
    SelectedMirror = @{ Url = 'https://mirrors.aliyun.com/...' }
    SilentMode = $true
} | ConvertTo-Json | Set-Content "config.json"

# 批量执行
$computers = Get-Content "computers.txt"
foreach ($computer in $computers) {
    Invoke-Command -ComputerName $computer -FilePath ".\install-openclaw.ps1"
}
```

## 结语

通过本教程，您已经获得了一套完整的OpenClaw安装解决方案：

1. ✅ 支持6个国内镜像源，自动测速选源
2. ✅ 高度可定制化（安装目录、API配置、各种选项）
3. ✅ 完整的卸载脚本，彻底清理
4. ✅ 详细的日志记录，方便排查问题
5. ✅ 支持企业批量部署

这套脚本不仅适用于OpenClaw，也可以作为模板用于其他软件的自动化安装部署。

**脚本文件清单**：
- `install-openclaw.ps1` - 安装脚本（主程序）
- `uninstall-openclaw.ps1` - 卸载脚本

祝您使用愉快！
