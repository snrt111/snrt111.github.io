---
title: windows安装docker
date: 2024-04-21 07:32:06
tags:
---

### 1.安装WSL

WSL指的是windows System for Linux

#### 1. 检查是否打开虚拟化设置（一般默认是开启的，但以防万一，还是检查一下）

1. 打开任务管理器（`Ctrl+Shift+Esc`）>>性能
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627223725385-1277211065.png)
   2.如果没有打开，打开BIOS系统设置，自行配置（不同电脑进入按键不同）

#### 2.配置文件

1. 打开windows的`启动或或关闭windows功能`
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627224036309-644332993.png)
2. 打开!`适用于linux的windows子系统`，系统会自动配置完成，可能会重启
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627224334429-870530824.png)

#### 3.下载WSL

1. 打开windows的`PowerShell`
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627224512241-741890783.png)
2. 运行`wsl --list --online`,选择要安装的版本
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627224859758-888096479.png)
3. 以Ubuntu-20.04为例,输入以下代码，等待下载完成

```shell
wsl --install -d Ubuntu-20.04
```

1. 下载完成后，可能会报错
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627225535551-1497364489.png)
   下载最新的wsl安装包,并运行安装，下载地址：https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi
2. 重新打开后，会让你输入用户名，以及密码，完成后即为安装成功
   **6. 之后的操作迁移wsl从C盘到其他盘，可以选择不进行操作**
3. 检查当前wsl是否在运行
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627230846016-659709016.png)
4. 如果显示的是running,则输入命令`wsl --shutdown`
   wsl --export Ubuntu G:\Virtual_Machines\Ubuntu.tar
   wsl --unregister Ubuntu
   wsl --import Ubuntu F:\Virtual_Machines G:\Virtual_Machines\Ubuntu.tar --version 2
5. 迁移

```lua
wsl --export Ubuntu-20.04 D:\Ubuntu.tar
wsl --unregister Ubuntu-20.04
wsl --import Ubuntu-20.04 D:\Ubuntu_2004 D:\Ubuntu.tar --version 2
```

1. 进行用户配置`<username>`是你前面注册的用户名

```lua
ubuntu.exe config --default-user <username>
```

#### 4. 下载Docker Desktop

1. 从官网下载Docker Desktop；https://www.docker.com/get-started/
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627225938679-691609522.png)
2. 建立存储路径软连接（Docker默认安装在C盘，如果想更得化，建立软链接）

```bash
mkdir "D:\Program Files\Docker"
mklink /j "C:\Program Files\Docker" "D:\Program Files\Docker"
```

1. 安装Docker Desktop,默认就行
2. 打开Docker Desktop,软件可能检测不到，然后自动退出，此时需要在打开时，快速点击右上角的设置按钮，并进行以下配置，并点击`apply`
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627231921741-1771734715.png)
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627232011813-1604727090.png)
3. 重新打开Docker Desktop,发现界面已改变
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627232230649-319362722.png)
4. 在`PowerShell`查看版本号`docker --version`和测试`docker run hello-world`
   ![image](https://img2022.cnblogs.com/blog/2891068/202206/2891068-20220627232615944-135304684.png)

至此在windows11中安装配置Docker成功