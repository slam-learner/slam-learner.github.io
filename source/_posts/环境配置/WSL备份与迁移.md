---
title: WSL备份与迁移
date: 2024-03-27 13:40:07
tags:
  - WSL
categories:
  - 环境配置
---

## 备份

终止正在运行的 wsl

```bash
wsl --shutdown
```

ubuntu 名字用 wsl -l -v 查看，这里以导出到 G 盘下，命名为 Ubuntu_bak.tar 为例

```typescript
wsl --export ubuntu-22.04 G:\Ubuntu_bak.tar
```

## 注销

注销之前的 wsl

```bash
wsl --unregister ubuntu-22.04
```

## 导入

```bash
wsl --import ubuntu-22.04 G:\WSL_Ubuntu G:\Ubuntu_bak.tar --version 2
```

## 升级 wsl 版本

如果之前的 wsl 版本为 wsl1，需要升级到 wsl2 的话，需要执行下面的命令：

```bash
wsl --set-version ubuntu-22.04 2
```

如果 powershell 中提示需要升级，那么需要执行：

```bash
wsl --update
```

设置默认 wsl 默认版本为 wsl2

```bash
wsl --set-default-version 2
```

查看安装版本

```bash
wsl -l -v
```

**参数解读：**
*Ubuntu-22.04*
· 实例名称，可以自己设置，设置后即为第 2 步输入 wsl -l -v 后显示的名称；
*G:\WSL_Ubuntu*
· 导入后的镜像及其相关数据存放路径；
*G:\wsl_nomatlab. Tar*
· 导入的备份，即第 3 步通过 wsl --export 导出的文件；
*–version 2*
· WSL 版本为2

**用户设置**
方法 1： 管理员身份运行 power shell, 执行 `ubuntu2004.exe config --default-user root` ,再在 power shell，执行以下命令，可以将 WSL2的默认系统设置为 Ubuntu-20.04。

```shell
wslconfig /setdefault Ubuntu-20.04
```

方法 2：修改 `/etc/wsl.conf` 文件

```bash
#Set the user when launching a distribution with WSL.

[user]
default=username
```

方法 3：在 windows terminal 中按下三角按钮打开设置, 并在命令行设置中进行修改， -d 参数修改发行版本，-u 修改用户。
![Windows Terminal设置](https://raw.githubusercontent.com/slam-learner/Figures/main/PicGo/202403271350893.png)

## 压缩空间

`WSL2` 本质上是虚拟机，所以 `Windows` 会自动创建 `vhdx` 后缀的虚拟磁盘文件作为存储。这个 `vhdx` 后缀的虚拟磁盘文件特点是可以自动扩容，但是一般不会自动缩容。一旦有很多文件把它“撑大”，即使把这些文件删除它也不会自动“缩小”。所以删除文件后还需要我们手动进行压缩才能释放磁盘空间。

首先在 PowerShell 中执行：

```powershell
# 关闭 wsl
wsl --shutdown

# 运行管理计算机驱动器的 DiskPart命令
diskpart
```

在新打开的 `DiskPart` 命令窗口中执行：

```powershell
# 选择虚拟磁盘文件
select vdisk file="D:\WSL\Ubuntu-18.04\ext4.vhdx" 

# 压缩文件
compact vdisk

# 压缩完毕后卸载磁盘（可能会有报错/警告，正常现象）
detach vdisk
```

## 需要开启或安装的功能/包

直接按 win 键搜索 “启用或关闭 Windows 功能”即可找到选项卡，开启以下功能（不开启也可以正常使用，不清楚具体影响）。

![![[启用或关闭Windows功能选项卡.png]]](https://raw.githubusercontent.com/slam-learner/Figures/main/PicGo/![[启用或关闭Windows功能选项卡.png]].png)
商店中安装 Ubuntu 18.04 LTS

![![[18.04LTS商店安装.png]]](https://raw.githubusercontent.com/slam-learner/Figures/main/PicGo/![[18.04LTS商店安装.png]].png)

## 参考资料

- [修改默认登陆用户](https://www.jianshu.com/p/aeef20207355)
- [WSL 使用与迁移]( https://blog.csdn.net/poena/article/details/106451847 )
- [wsl 版本升级]( https://blog.csdn.net/m0_60634555/article/details/129699609 )
