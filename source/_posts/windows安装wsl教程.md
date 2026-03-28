---
title: Windows安装OpenClaw保姆级教程
date: 2026-03-24 10:00:00
categories:
  - 开发工具
tags:
  - 工具
---

在 Windows 10 专业版上安装 WSL（Windows Subsystem for Linux）非常简单。微软提供了一个一键安装的命令，这是最推荐的方法。

### 🚀 快速安装（推荐）

这是最简单快捷的安装方式，适用于 Windows 10 版本 2004 及更高版本（内部版本 19041+）。

1. **以管理员身份打开 PowerShell**

   - 在开始菜单搜索 "PowerShell"。
   - 右键点击 "Windows PowerShell"，选择“以管理员身份运行”。

2. **运行安装命令**

   - 在打开的 PowerShell 窗口中，输入以下命令并按回车：

     ```
     1wsl --install
     ```

   - 这个命令会自动完成以下操作：

     - 启用“适用于 Linux 的 Windows 子系统”和“虚拟机平台”这两个必需的 Windows 功能。
     - 下载并安装 WSL 2 的 Linux 内核。
     - 将 WSL 2 设置为默认版本。
     - 从 Microsoft Store 下载并安装默认的 Linux 发行版（通常是 Ubuntu）。

3. **重启计算机**

   - 命令执行完成后，根据提示重启你的电脑以使更改生效。

4. **完成初始化设置**

   - 重启后，系统可能会自动打开一个终端窗口，或者你可以从开始菜单启动刚安装的 Ubuntu。
   - 首次启动时，会提示你创建一个新的 UNIX 用户名和密码。
     - **用户名**：可以是任意名称，不必与你的 Windows 用户名一致。
     - **密码**：输入时屏幕上不会显示字符，这是 Linux 的正常安全特性，请放心输入并牢记。

设置完成后，你就可以开始使用 WSL 了。

### 🛠️ 手动安装（备选方案）

如果你的 Windows 10 版本较旧，或者 `wsl --install` 命令遇到问题，可以按照以下步骤手动安装。

1. **启用 WSL 和虚拟机平台功能**

   - 以管理员身份打开 PowerShell，依次运行以下两条命令：

     ```
     1dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
     2dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
     ```

2. **重启计算机**

   - 运行完上述命令后，重启你的电脑。

3. **下载并安装 WSL 2 内核更新包**

   - 从微软官方链接下载适用于 x64 计算机的更新包：
     [WSL2 Linux 内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)
   - 下载完成后，双击运行安装程序。

4. **将 WSL 2 设置为默认版本**

   - 再次以管理员身份打开 PowerShell，运行以下命令：

     ```
     1wsl --set-default-version 2
     ```

5. **安装 Linux 发行版**

   - 打开 Microsoft Store，搜索你喜欢的 Linux 发行版（如 "Ubuntu 22.04 LTS"、"Debian" 等），然后点击“获取”或“安装”。
   - 安装完成后，从开始菜单启动它，并按照“快速安装”中的第4步完成初始化设置。