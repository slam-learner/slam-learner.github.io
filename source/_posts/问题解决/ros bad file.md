---
title: ros bad file
date: 2024-03-13 15:06:04
tags:
  - 问题解决
  - ROS
  - WSL
categories:
  - 问题解决
---

## 问题描述

在WSL的一个ROS工作空间对项目进行编译的时候出现了以下错误信息提示：

```bash
/home/ydx/code/CPP/download_codes/autonomous_yhs_fr07/build/catkin_generated/env_cached.sh: 12: export: Files/Microsoft: bad variable name
CMake Error at /opt/ros/melodic/share/catkin/cmake/safe_execute_process.cmake:11 (message):
  
  execute_process(/home/ydx/code/CPP/download_codes/autonomous_yhs_fr07/build/catkin_generated/env_cached.sh
  "/usr/bin/python2" "/usr/bin/empy" "--raw-errors" "-F"
  "/home/ydx/code/CPP/download_codes/autonomous_yhs_fr07/build/catkin_generated/order_packages.py"
  "-o"
  "/home/ydx/code/CPP/download_codes/autonomous_yhs_fr07/build/catkin_generated/order_packages.cmake"
  "/opt/ros/melodic/share/catkin/cmake/em/order_packages.cmake.em") returned
  error code 2
Call Stack (most recent call first):
  /opt/ros/melodic/share/catkin/cmake/em_expand.cmake:25 (safe_execute_process)
  /opt/ros/melodic/share/catkin/cmake/catkin_workspace.cmake:35 (em_expand)
  CMakeLists.txt:69 (catkin_workspace)
```

由第一行的`export: Files/Microsoft: bad variable name`可以看出是某个环境变量中有类似`Program Files/Microsoft`这样的路径导致的错误。显然这和WSL的路径有关。通过在网上查找资料，发现了[askubuntu.com上面的一个帖子](https://askubuntu.com/questions/1354999/bad-variable-name-error-on-wsl)有一个类似的问题描述，这里参照第二个回答，先在`/etc/wsl.conf`文件添加了下面两行：

```bash
[interop]
appendWindowsPath = false
```

如果在诸如`~/.bashrc`或者`~/.zshrc`文件中添加了类似`export PATH=$PATH:/mnt/c/Windows/System32`这样的路径，也需要将这些路径注释掉。

然后在`PowerShell`中执行`wsl --shutdown`命令彻底关闭WSL后重启WSL，重新编译时不再出现这个错误。在编译完成后，需要将之前完成的更改恢复，否则诸如VSCode等工具将无法正常使用。
