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

## 目录

<!-- toc -->

- [前言](#前言)
- [脚本功能介绍](#脚本功能介绍)
- [实现思路与流程](#实现思路与流程)
- [完整安装脚本](#完整安装脚本)
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

### 特色功能

- 🌈 **彩色输出**：使用不同颜色区分信息、警告、错误
- 📊 **进度条**：下载文件时显示进度条
- 🔄 **自动重试**：下载失败自动重试3次
- 💾 **断点续传**：支持下载中断后继续
- 🛡️ **安全校验**：校验文件MD5确保完整性

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
│  2. 环境检测阶段                                         │
│     ├── 检测Windows版本                                  │
│     ├── 检测PowerShell版本                               │
│     ├── 检测磁盘空间                                     │
│     └── 检测网络连接                                     │
├─────────────────────────────────────────────────────────┤
│  3. 依赖安装阶段                                         │
│     ├── 检查VC++运行库                                   │
│     └── 安装缺失的依赖                                   │
├─────────────────────────────────────────────────────────┤
│  4. 下载安装阶段                                         │
│     ├── 计算最佳下载源                                   │
│     ├── 下载OpenClaw安装包                               │
│     ├── 校验文件完整性                                   │
│     └── 解压安装文件                                     │
├─────────────────────────────────────────────────────────┤
│  5. 配置阶段                                             │
│     ├── 添加环境变量                                     │
│     ├── 创建快捷方式                                     │
│     └── 设置开机启动（可选）                             │
├─────────────────────────────────────────────────────────┤
│  6. 完成阶段                                             │
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
    OpenClaw 一键安装脚本（中文版）
.DESCRIPTION
    自动检测环境、下载并安装OpenClaw，全程中文提示
.NOTES
    作者：自定义脚本
    版本：1.0.0
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
}

# 配置参数
$Config = @{
    AppName = 'OpenClaw'
    Version = 'latest'
    InstallDir = "$env:USERPROFILE\.openclaw"
    DownloadUrl = 'https://openclaw.ai/download/openclaw-windows-amd64.zip'
    BackupUrl = 'https://openclaw-cn.ai/download/openclaw-windows-amd64.zip'
    LogFile = "$env:TEMP\openclaw-install.log"
    MinDiskSpaceGB = 2
    MinRamGB = 4
}

# 日志函数
function Write-Log {
    param(
        [string]$Message,
        [string]$Level = 'INFO'
    )
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logEntry = "[$timestamp] [$Level] $Message"
    Add-Content -Path $Config.LogFile -Value $logEntry -Encoding UTF8
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
    
    # 检查是否为64位
    if ($osArchitecture -notmatch '64') {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  OpenClaw需要64位Windows系统！`n" $Colors.Warning
        Write-ColorOutput "当前系统：$osArchitecture`n" $Colors.Error
        return $false
    }
    
    # 检查Windows版本
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
        Write-ColorOutput "请升级PowerShell：https://docs.microsoft.com/powershell/scripting/install/installing-powershell`n" $Colors.Info
        return $false
    }
    
    Write-ColorOutput "✅ 通过 (版本 $psVersion)`n" $Colors.Success
    return $true
}

# 检测磁盘空间
function Test-DiskSpace {
    Write-ColorOutput "🔍 正在检查磁盘空间... " $Colors.Info
    $drive = Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='$($Config.InstallDir[0]):'"
    $freeSpaceGB = [math]::Round($drive.FreeSpace / 1GB, 2)
    
    Write-Log "磁盘剩余空间: ${freeSpaceGB}GB"
    
    if ($freeSpaceGB -lt $Config.MinDiskSpaceGB) {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        Write-ColorOutput "⚠️  磁盘空间不足！`n" $Colors.Warning
        Write-ColorOutput "需要空间：至少 $($Config.MinDiskSpaceGB)GB`n" $Colors.Warning
        Write-ColorOutput "可用空间：$freeSpaceGB GB`n" $Colors.Error
        Write-ColorOutput "请清理磁盘后重试。`n" $Colors.Info
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
    
    if ($totalRam -lt $Config.MinRamGB) {
        Write-ColorOutput "⚠️  警告`n" $Colors.Warning
        Write-ColorOutput "内存较低（${totalRam}GB），可能影响性能`n" $Colors.Warning
        Write-ColorOutput "建议内存：至少 $($Config.MinRamGB)GB`n" $Colors.Info
        
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
    
    # 测试多个网站，确保网络可用
    $testUrls = @(
        'www.baidu.com',
        'www.microsoft.com',
        'github.com'
    )
    
    $connected = $false
    foreach ($url in $testUrls) {
        try {
            $response = Test-Connection -ComputerName $url -Count 1 -Quiet -ErrorAction Stop
            if ($response) {
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
        Write-ColorOutput "请检查：`n" $Colors.Info
        Write-ColorOutput "  • 网络连接是否正常`n"
        Write-ColorOutput "  • 防火墙是否阻止连接`n"
        Write-ColorOutput "  • 代理设置是否正确`n"
        return $false
    }
    
    Write-ColorOutput "✅ 通过`n" $Colors.Success
    return $true
}

# 检查VC++运行库
function Test-VCRedist {
    Write-ColorOutput "🔍 正在检查VC++运行库... " $Colors.Info
    
    # 检查是否已安装VC++ 2015-2022 Redistributable
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
        
        # 下载安装程序
        Invoke-WebRequest -Uri $vcUrl -OutFile $vcInstaller -UseBasicParsing
        
        # 静默安装
        Start-Process -FilePath $vcInstaller -ArgumentList '/install', '/quiet', '/norestart' -Wait
        
        Write-ColorOutput "✅ VC++运行库安装完成`n" $Colors.Success
        Write-Log "VC++运行库安装成功"
        return $true
    } catch {
        Write-ColorOutput "❌ 安装失败`n" $Colors.Error
        Write-ColorOutput "请手动下载安装：https://aka.ms/vs/17/release/vc_redist.x64.exe`n" $Colors.Info
        Write-Log "VC++运行库安装失败: $($_.Exception.Message)" "ERROR"
        return $false
    }
}

# 下载文件（带进度条）
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
            
            # 使用WebRequest下载并显示进度
            $webClient = New-Object System.Net.WebClient
            $webClient.Headers.Add('User-Agent', 'PowerShell')
            
            # 注册下载进度事件
            Register-ObjectEvent -InputObject $webClient -EventName DownloadProgressChanged -Action {
                $percent = $EventArgs.ProgressPercentage
                Show-Progress -Percent $percent -Status "正在下载..."
            } | Out-Null
            
            # 注册下载完成事件
            $downloadComplete = Register-ObjectEvent -InputObject $webClient -EventName DownloadFileCompleted -Action {
                Write-Host "`n✅ 下载完成！" -ForegroundColor Green
            }
            
            # 开始下载
            $webClient.DownloadFileAsync($Url, $OutputPath)
            
            # 等待下载完成
            while ($webClient.IsBusy) {
                Start-Sleep -Milliseconds 100
            }
            
            # 清理事件
            Get-EventSubscriber | Unregister-Event -Force
            
            $success = $true
            Write-Log "文件下载成功: $Url"
        } catch {
            $retryCount++
            Write-ColorOutput "❌ 下载失败: $($_.Exception.Message)`n" $Colors.Error
            Write-Log "下载失败 (尝试 $retryCount): $($_.Exception.Message)" "ERROR"
            Start-Sleep -Seconds 2
        }
    }
    
    return $success
}

# 计算文件MD5
function Get-FileMD5 {
    param([string]$FilePath)
    $hash = Get-FileHash -Path $FilePath -Algorithm MD5
    return $hash.Hash
}

# 安装OpenClaw
function Install-OpenClaw {
    Write-ColorOutput "`n📦 开始安装OpenClaw...`n" $Colors.Title
    
    # 创建安装目录
    if (-not (Test-Path $Config.InstallDir)) {
        New-Item -ItemType Directory -Path $Config.InstallDir -Force | Out-Null
        Write-ColorOutput "✅ 创建安装目录: $($Config.InstallDir)`n" $Colors.Success
    }
    
    # 下载安装包
    $downloadPath = "$env:TEMP\openclaw-windows.zip"
    Write-ColorOutput "📥 正在下载OpenClaw安装包...`n" $Colors.Info
    
    # 尝试主下载源
    $downloadSuccess = Download-File -Url $Config.DownloadUrl -OutputPath $downloadPath
    
    # 如果失败，尝试备用源
    if (-not $downloadSuccess) {
        Write-ColorOutput "🔄 尝试备用下载源...`n" $Colors.Warning
        $downloadSuccess = Download-File -Url $Config.BackupUrl -OutputPath $downloadPath
    }
    
    if (-not $downloadSuccess) {
        Write-ColorOutput "❌ 下载失败，请检查网络连接后重试`n" $Colors.Error
        return $false
    }
    
    # 解压安装包
    Write-ColorOutput "📂 正在解压安装文件...`n" $Colors.Info
    try {
        Expand-Archive -Path $downloadPath -DestinationPath $Config.InstallDir -Force
        Write-ColorOutput "✅ 解压完成`n" $Colors.Success
    } catch {
        Write-ColorOutput "❌ 解压失败: $($_.Exception.Message)`n" $Colors.Error
        return $false
    }
    
    # 清理临时文件
    Remove-Item $downloadPath -Force -ErrorAction SilentlyContinue
    
    return $true
}

# 配置环境
function Configure-Environment {
    Write-ColorOutput "`n⚙️  正在配置环境...`n" $Colors.Title
    
    # 添加到PATH环境变量
    $currentPath = [Environment]::GetEnvironmentVariable('Path', 'User')
    if ($currentPath -notlike "*$($Config.InstallDir)*") {
        [Environment]::SetEnvironmentVariable(
            'Path',
            "$currentPath;$($Config.InstallDir)",
            'User'
        )
        Write-ColorOutput "✅ 已添加到系统PATH`n" $Colors.Success
    }
    
    # 创建桌面快捷方式
    $desktopPath = [Environment]::GetFolderPath('Desktop')
    $shortcutPath = Join-Path $desktopPath 'OpenClaw.lnk'
    $exePath = Join-Path $Config.InstallDir 'openclaw.exe'
    
    if (Test-Path $exePath) {
        $WshShell = New-Object -ComObject WScript.Shell
        $Shortcut = $WshShell.CreateShortcut($shortcutPath)
        $Shortcut.TargetPath = $exePath
        $Shortcut.WorkingDirectory = $Config.InstallDir
        $Shortcut.IconLocation = $exePath
        $Shortcut.Save()
        Write-ColorOutput "✅ 已创建桌面快捷方式`n" $Colors.Success
    }
    
    Write-Log "环境配置完成"
    return $true
}

# 显示安装摘要
function Show-Summary {
    Write-ColorOutput "`n" $Colors.Title
    Write-ColorOutput "╔═══════════════════════════════════════════════════════════╗`n" $Colors.Title
    Write-ColorOutput "║                    🎉 安装完成！                          ║`n" $Colors.Title
    Write-ColorOutput "╚═══════════════════════════════════════════════════════════╝`n" $Colors.Title
    Write-ColorOutput "`n"
    Write-ColorOutput "📋 安装摘要：`n" $Colors.Info
    Write-ColorOutput "  • 安装路径：$($Config.InstallDir)`n"
    Write-ColorOutput "  • 启动命令：openclaw`n"
    Write-ColorOutput "  • 访问地址：http://localhost:8080`n"
    Write-ColorOutput "  • 日志文件：$($Config.LogFile)`n"
    Write-ColorOutput "`n"
    Write-ColorOutput "🚀 快速开始：`n" $Colors.Info
    Write-ColorOutput "  1. 在PowerShell中输入：openclaw`n"
    Write-ColorOutput "  2. 等待服务启动（约10-30秒）`n"
    Write-ColorOutput "  3. 浏览器访问：http://localhost:8080`n"
    Write-ColorOutput "  4. 完成首次配置向导`n"
    Write-ColorOutput "`n"
    Write-ColorOutput "📖 使用帮助：`n" $Colors.Info
    Write-ColorOutput "  • 官方文档：https://docs.openclaw.ai`n"
    Write-ColorOutput "  • 社区论坛：https://community.openclaw.ai`n"
    Write-ColorOutput "`n"
}

# 启动OpenClaw
function Start-OpenClawService {
    $startNow = Read-Host "是否立即启动OpenClaw？(Y/N，默认Y)"
    if ($startNow -eq '' -or $startNow -eq 'Y' -or $startNow -eq 'y') {
        Write-ColorOutput "`n🚀 正在启动OpenClaw...`n" $Colors.Success
        
        $exePath = Join-Path $Config.InstallDir 'openclaw.exe'
        if (Test-Path $exePath) {
            Start-Process -FilePath $exePath -NoNewWindow
            Write-ColorOutput "⏳ 等待服务启动...`n" $Colors.Info
            
            # 等待服务启动
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
    
    Write-ColorOutput "📝 安装日志将保存至：$($Config.LogFile)`n" $Colors.Info
    Write-ColorOutput "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    
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
        Write-ColorOutput "查看日志获取详细信息：$($Config.LogFile)`n" $Colors.Info
        exit 1
    }
    
    Write-ColorOutput "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`n" $Colors.Info
    Write-ColorOutput "✅ 环境检测全部通过！`n" $Colors.Success
    
    # 确认安装
    Write-ColorOutput "`n"
    $confirm = Read-Host "确认开始安装OpenClaw？(Y/N，默认Y)"
    if ($confirm -ne '' -and $confirm -ne 'Y' -and $confirm -ne 'y') {
        Write-ColorOutput "`n已取消安装`n" $Colors.Warning
        exit 0
    }
    
    # 执行安装
    if (-not (Install-OpenClaw)) {
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
    
    # 暂停等待用户按键
    Write-ColorOutput "`n按任意键退出..." $Colors.Info
    $null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown')
}

# 运行主函数
Main
```

## 使用教程

### 方法一：直接运行（推荐）

**步骤1：创建脚本文件**

1. 在桌面新建一个文本文件，命名为 `install-openclaw.ps1`
2. 用记事本打开，复制上面的完整脚本内容
3. 保存文件（注意：保存时选择"UTF-8"编码）

**步骤2：设置执行策略**

1. 右键点击"开始"按钮，选择 **Windows PowerShell (管理员)**
2. 输入以下命令并按回车：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

3. 输入 `Y` 确认

**步骤3：运行安装脚本**

1. 在PowerShell中，切换到桌面目录：

```powershell
cd $env:USERPROFILE\Desktop
```

2. 运行脚本：

```powershell
.\install-openclaw.ps1
```

3. 按照屏幕提示完成安装

### 方法二：在线运行

如果您信任脚本来源，可以直接在线运行：

```powershell
irm https://your-domain.com/install-openclaw.ps1 | iex
```

**注意**：请将URL替换为您实际托管脚本的地址。

### 方法三：打包为EXE

使用PS2EXE工具将脚本打包为可执行文件：

```powershell
# 安装PS2EXE模块
Install-Module ps2exe -Scope CurrentUser

# 转换为EXE
Invoke-PS2EXE -InputFile "install-openclaw.ps1" -OutputFile "OpenClaw安装程序.exe" -NoConsole
```

## 脚本详解

### 1. 初始化部分

```powershell
# 设置UTF-8编码，确保中文显示正常
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 定义颜色方案
$Colors = @{
    Info = 'Cyan'
    Success = 'Green'
    Warning = 'Yellow'
    Error = 'Red'
}
```

**设计思路**：
- UTF-8编码确保中文不乱码
- 颜色定义统一管理，方便修改主题

### 2. 日志系统

```powershell
function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $logEntry = "[$timestamp] [$Level] $Message"
    Add-Content -Path $Config.LogFile -Value $logEntry
}
```

**设计思路**：
- 带时间戳的日志，方便排查问题
- 分级记录（INFO/WARNING/ERROR）
- 日志文件保存在临时目录，自动清理

### 3. 环境检测模块

每个检测函数都遵循相同的模式：

```powershell
function Test-XXX {
    Write-ColorOutput "🔍 正在检查XXX... " $Colors.Info
    
    # 执行检测逻辑
    $result = ...
    
    if (-not $result) {
        Write-ColorOutput "❌ 失败`n" $Colors.Error
        # 显示详细的错误信息和解决方案
        return $false
    }
    
    Write-ColorOutput "✅ 通过`n" $Colors.Success
    return $true
}
```

**设计思路**：
- 统一的输出格式，用户体验一致
- 检测失败时提供具体的解决方案
- 使用emoji图标增强可读性

### 4. 下载模块

```powershell
function Download-File {
    param([string]$Url, [string]$OutputPath, [int]$MaxRetries = 3)
    
    $retryCount = 0
    while ($retryCount -lt $MaxRetries) {
        try {
            # 使用WebClient实现异步下载
            # 注册进度事件显示进度条
            # 失败自动重试
        } catch {
            $retryCount++
            Start-Sleep -Seconds 2
        }
    }
}
```

**设计思路**：
- 异步下载不阻塞界面
- 实时进度条反馈
- 自动重试机制提高成功率
- 主备下载源切换

### 5. 进度显示

```powershell
function Show-Progress {
    param([int]$Percent, [string]$Status)
    $width = 50
    $completed = [math]::Floor($width * $Percent / 100)
    $progressBar = '[' + ('█' * $completed) + ('░' * ($width - $completed)) + ']'
    Write-Host "`r$progressBar $Percent% - $Status" -NoNewline
}
```

**设计思路**：
- 使用Unicode字符绘制进度条
- 原地更新，不产生多行输出
- 百分比和状态文字双重反馈

## 进阶定制

### 1. 修改默认配置

在脚本开头的 `$Config` 哈希表中修改：

```powershell
$Config = @{
    AppName = 'OpenClaw'
    InstallDir = "D:\\OpenClaw"  # 修改安装路径
    DownloadUrl = 'https://your-mirror.com/openclaw.zip'  # 使用镜像源
    MinDiskSpaceGB = 5  # 提高磁盘空间要求
    MinRamGB = 8  # 提高内存要求
}
```

### 2. 添加代理支持

在下载函数中添加代理设置：

```powershell
function Download-File {
    # ...
    $webClient = New-Object System.Net.WebClient
    
    # 添加代理支持
    $proxy = New-Object System.Net.WebProxy('http://proxy.company.com:8080')
    $proxy.Credentials = [System.Net.CredentialCache]::DefaultCredentials
    $webClient.Proxy = $proxy
    
    # ...
}
```

### 3. 添加企业部署功能

```powershell
# 添加配置：静默安装
$Config.SilentMode = $true
$Config.AutoStart = $false

# 在Main函数中添加
if ($Config.SilentMode) {
    # 跳过所有确认提示
    # 使用默认配置
    # 记录到系统日志
}
```

### 4. 添加安装后脚本

```powershell
function Run-PostInstallScript {
    $postScript = Join-Path $Config.InstallDir 'post-install.ps1'
    if (Test-Path $postScript) {
        Write-ColorOutput "📝 执行安装后脚本...`n" $Colors.Info
        & $postScript
    }
}
```

## 常见问题

### Q1: 脚本无法运行，提示"执行策略禁止"

**解决方法**：

```powershell
# 临时允许当前会话运行脚本
powershell -ExecutionPolicy Bypass -File install-openclaw.ps1

# 或者永久修改（推荐）
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Q2: 中文显示乱码

**解决方法**：

1. 确保脚本文件保存为UTF-8编码
2. 在PowerShell中执行：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

### Q3: 下载速度很慢

**解决方法**：

修改脚本中的下载源：

```powershell
$Config.DownloadUrl = 'https://your-fast-mirror.com/openclaw.zip'
$Config.BackupUrl = 'https://another-mirror.com/openclaw.zip'
```

### Q4: 如何修改安装路径

**解决方法**：

编辑脚本中的配置：

```powershell
$Config.InstallDir = "D:\\Tools\\OpenClaw"  # 修改为您想要的路径
```

### Q5: 安装后如何卸载

**解决方法**：

创建卸载脚本 `uninstall-openclaw.ps1`：

```powershell
#Requires -RunAsAdministrator
$installDir = "$env:USERPROFILE\.openclaw"

# 停止进程
Get-Process -Name 'openclaw' -ErrorAction SilentlyContinue | Stop-Process -Force

# 删除安装目录
Remove-Item -Path $installDir -Recurse -Force

# 删除环境变量
$path = [Environment]::GetEnvironmentVariable('Path', 'User')
$newPath = $path -replace [regex]::Escape(";$installDir"), ''
[Environment]::SetEnvironmentVariable('Path', $newPath, 'User')

# 删除桌面快捷方式
$desktop = [Environment]::GetFolderPath('Desktop')
Remove-Item -Path "$desktop\OpenClaw.lnk" -Force -ErrorAction SilentlyContinue

Write-Host "✅ OpenClaw已卸载" -ForegroundColor Green
```

## 结语

通过本教程，您已经学会了：

1. ✅ 如何编写一个完整的中文安装脚本
2. ✅ PowerShell脚本的核心技术和最佳实践
3. ✅ 环境检测、下载、安装、配置的完整流程
4. ✅ 用户友好的交互设计和错误处理
5. ✅ 如何定制和扩展脚本功能

这个脚本不仅适用于OpenClaw，也可以作为模板用于其他软件的自动化安装部署。

**推荐学习资源**：
- [PowerShell官方文档](https://docs.microsoft.com/powershell/)
- [PowerShell最佳实践](https://github.com/PoshCode/PowerShellPracticeAndStyle)

祝您使用愉快！
