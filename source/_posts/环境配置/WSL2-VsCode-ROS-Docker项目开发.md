---
title: WSL2-VsCode-ROS-Docker项目开发
date: 2024-04-09 22:05:11
tags:
  - VS Code
  - docker
  - WSL2
categories:
  - 环境配置
---

## 介绍

在 Windows 系统上使用 WSL2 + VS Code + ROS + Docker 进行项目开发。

WSL2 + VS Code是此前一直使用的开发方式，只要使用VS Code的WSL插件，就可以在WSL2中进行开发，而不用在Windows系统中安装编译器和调试器。在一般的C++项目开发中，运行调试项目可以通过设置`.vscode/launch.jso`n，`.vscode/tasks.json`，`.vscode/c_cpp_properties.json`等文件来实现。但是在开发ROS项目时，需要在VSC中安装ROS插件，这样才能在VSC中运行ROS项目。并且使用ROS插件提供的设置来运行ROS项目。这个插件有一些bug，有时会出现一些奇怪的问题。另外，在运行`ros_motion_planning`项目时，在`ubuntu-18.04`下始终无法编译通过，而在`ubuntu-20.04`下运行还需要设置很多东西，恰巧该项目提供了`dockerfile`，因此使用 WSL2 + VS Code + ROS + Docker 进行项目开发。这里对于遇到的问题及解决方案进行记录。有一些具有很多教程的步骤，在这里不再赘述。比如安装WSL2，安装VS Code，安装ROS，Windows下安装docker并使用WSL2作为后端引擎等。

## docker的基本命令
  
```bash
# 查看所有容器
docker ps -a
# 查看所有镜像
docker images
# 删除容器
docker rm -f <container_id>
# 删除镜像
docker rmi <image_id>
# 进入容器
docker exec -it <container_id> /bin/bash
# 退出容器
exit
# 构建镜像
docker build -t <image_name> .
# 运行容器
docker run -it --name <container_name> <image_name> /bin/bash
# 删除镜像
docker image prune
# 删除不再使用的数据卷
docker volume prune
# 删除build cache
docker builder prune
# 删除所有未使用的资源
docker system prune
# 查看docker占用空间
docker system df
```

## 项目docker镜像构建

ros_motion_planning的项目结构如下：

```bash
$ cd ros_motion_planning
$ tree -d -L 3
.
├── assets
├── data
├── docker
│   └── noetic
├── docs
├── package
│   ├── osqp
│   │   ├── configure
│   │   ├── docs
│   │   ├── examples
│   │   ├── include
│   │   ├── lin_sys
│   │   ├── site
│   │   ├── src
│   │   └── tests
│   └── osqp-eigen
│       ├── cmake
│       ├── docs
│       ├── example
│       ├── include
│       ├── src
│       └── tests
├── scripts
└── src
    ├── core
    │   ├── curve_generation
    │   ├── global_planner
    │   ├── local_planner
    │   ├── trajectory_optimization
    │   └── utils
    ├── sim_env
    │   ├── config
    │   ├── launch
    │   ├── maps
    │   ├── meshes
    │   ├── models
    │   ├── rviz
    │   ├── scripts
    │   ├── urdf
    │   └── worlds
    ├── third_party
    │   ├── dynamic_rviz_config
    │   ├── dynamic_xml_config
    │   ├── gazebo_plugins
    │   ├── map_plugins
    │   └── rviz_plugins
    └── user_config
```

项目提供的dockerfile如下：

```dockerfile
# Use the official ROS Noetic base image
FROM osrf/ros:noetic-desktop-full

# Set the working directory
WORKDIR /project

# Install additional dependencies if needed
# psmisc(用于进程查看)，libxml2、libxslt(用于 XML/XSLT 处理)，python-is-python3(将 python3 设置为默认 python 版本)
RUN apt-get update \
    && apt-get -y --no-install-recommends install \
    git \
    gcc \
    vim \
    psmisc \
    libxml2-dev \
    libxslt-dev \
    python3 \
    python3-pip \ 
    python-is-python3\
    ros-noetic-amcl \
    ros-noetic-base-local-planner \
    ros-noetic-map-server \
    ros-noetic-move-base \
    ros-noetic-navfn

# python packages
RUN pip3 install setuptools && pip3 install catkin-tools

# bash
RUN echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc

# Copy the project into the container
COPY . /project

# catkin build
RUN /bin/bash -c '. /opt/ros/noetic/setup.bash; catkin_make'
```

在执行软件更新及安装时，由于网络问题，可能会更新较慢或者失败，可以在dockerfile中添加中科大的源，如下：

```dockerfile
RUN sed -i 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
```

具体来说:

- `sed` 是一个流编辑器,常用于文本替换操作。
- `-i` 选项指示 `sed` 直接修改文件内容,而不是输出到标准输出。
- 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' 是一个替换命令,其中: `s` 是替换命令。`archive.ubuntu.com` 是要被替换的原始字符串。`mirrors.ustc.edu.cn` 是要替换成的新字符串。`g` 表示在整个文件中进行全局替换,而不仅仅是替换第一次出现的地方。

总结来说，以上命令可将 `/etc/apt/sources.list` 文件中所有出现的 `archive.ubuntu.com` 都替换为 `mirrors.ustc.edu.cn`。

增加了中科大源后，构建镜像时，速度会快很多，镜像构建的命令如下：

```bash
cd docker/noetic
docker build -t ros-motion-planning:noetic --no-cache -f ./Dockerfile ../../
```

构建在进行到最后一步的`catkin_make`时，出现以下错误：

```bash
9.440 CMake Error at core/local_planner/mpc_planner/CMakeLists.txt:19 (find_package):
9.440   By not providing "FindOsqpEigen.cmake" in CMAKE_MODULE_PATH this project
9.440   has asked CMake to find a package configuration file provided by
9.440   "OsqpEigen", but CMake did not find one.
9.440 
9.440   Could not find a package configuration file provided by "OsqpEigen" with
9.440   any of the following names:
9.440 
9.440     OsqpEigenConfig.cmake
9.440     osqpeigen-config.cmake
9.440 
9.440   Add the installation prefix of "OsqpEigen" to CMAKE_PREFIX_PATH or set
9.440   "OsqpEigen_DIR" to a directory containing one of the above files.  If
9.440   "OsqpEigen" provides a separate development package or SDK, be sure it has
9.440   been installed.
9.440 
9.440 
9.448 -- Configuring incomplete, errors occurred!
```

显然是由于缺少`OsqpEigen`库导致的错误，查看项目的`README.md`文件，可看到以下关于`OSQP`和`OsqpEigen`安装的部分：

```bash
# OSQP
git clone -b release-0.6.3 --recursive https://github.com/oxfordcontrol/osqp
cd osqp && mkdir build && cd build
cmake .. -DBUILD_SHARED_LIBS=ON
make -j6
sudo make install
sudo cp /usr/local/include/osqp/* /usr/local/include

# OsqpEigen
git clone https://github.com/robotology/osqp-eigen.git
cd osqp-eigen && mkdir build && cd build
cmake ..
make
sudo make install
```

所以可以参考以上内容，在构建时增加`OSQP`和`OsqpEigen`的安装，首先新建一个`package`目录用于存放`OSQP`和`OsqpEigen`的源文件，然后在`scripts`目录下新建一个`install_osqp.sh`文件，内容如下：

```bash
#!/bin/bash
set -e

# Install osqp
cd /project/package/osqp
mkdir build && cd build
cmake .. -DBUILD_SHARED_LIBS=ON
make -j$(nproc)
make install

# clean up
cd /project/package/osqp
rm -rf build

# Install osqp-eigen
cd /project/package/osqp-eigen
mkdir build && cd build
cmake ..
make -j$(nproc)
make install

# clean up
cd /project/package/osqp-eigen
rm -rf build
```

在`dockerfile`中增加以下内容：

```dockerfile
# Install osqp and osqp-eigen
RUN /project/scripts/install_osqp.sh
```

出现以下错误：

```bash
=> ERROR [ 9/10] RUN /project/scripts/install_osqp.sh 0.5s
------
> [ 9/10] RUN /project/scripts/install_osqp.sh:
0.471 /project/scripts/install_osqp.sh: line 5: cd: /project/package/osqp: No such file or directory
------
```

仔细排查后，发现这是由于项目的`.dockerignore`文件导致的，.dockerignore 文件用于排除不需要复制到 Docker 镜像中的文件和目录, 该项目的`.dockerignore`文件如下：

```bash
*

!assets
!docker
!docs
!package
!scripts
!src
!Doxyfile
!LICENSE
!README.md
!.clang-format
```

其中`*`表示排除所有文件和目录，`!`表示不排除，所以需要在`.dockerignore`文件中添加`!package`，然后重新构建镜像，此时出现以下错误：

```bash
27.27 Scanning dependencies of target mpc_planner
27.30 [89%] Building CXX object core/local_planner/mpc_planner/CMakeFiles/mpc_planner.dir/src/mpc_planner.cpp.o
27.52 In file included from /usr/local/include/OsqpEigen/Compat.hpp:12,
27.52 from /usr/local/include/OsqpEigen/Constants.hpp:11,
27.52 from /usr/local/include/OsqpEigen/OsqpEigen.h:10,
27.52 from /project/src/core/local_planner/mpc_planner/src/mpc_planner.cpp:19:
27.52 /usr/local/include/osqp.h:9:11: fatal error: types.h: No such file or directory
27.52 9 | # include "types.h"
27.52 | ^~~~~~~~~
27.52 compilation terminated.
27.52 make[2]: *** [core/local_planner/mpc_planner/CMakeFiles/mpc_planner.dir/build.make:63: core/local_planner/mpc_planner/CMakeFiles/mpc_planner.dir/src/mpc_planner.cpp.o] Error 1
27.52 make[1]: *** [CMakeFiles/Makefile2:8368: core/local_planner/mpc_planner/CMakeFiles/mpc_planner.dir/all] Error 2
27.52 make[1]: *** Waiting for unfinished jobs....
```

尝试将`OSQP`和`OsqpEigen`的源文件复制到`/usr/local/include`目录下，但是似乎没有作用，因此放弃了这种方式，而是直接在`dockerfile`中将`OSQP`的include路径添加到了环境变量中，此后构建成功，完整的修改后的`dockerfile`如下：

```dockerfile
# Use the official ROS Noetic base image
FROM osrf/ros:noetic-desktop-full

# Set the working directory
WORKDIR /project

RUN sed -i 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
# Install additional dependencies if needed
RUN apt-get update \
    && apt-get -y --no-install-recommends install \
    git \
    gcc \
    vim \
    psmisc \
    libxml2-dev \
    libxslt-dev \
    python3 \
    python3-pip \ 
    python-is-python3\
    ros-noetic-amcl \
    ros-noetic-base-local-planner \
    ros-noetic-map-server \
    ros-noetic-move-base \
    ros-noetic-navfn

# python packages
RUN pip3 install setuptools && pip3 install catkin-tools

# bash
RUN echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc

# Copy the project into the container
COPY . /project

# Install osqp and osqposqp-eigen
RUN /project/scripts/install_osqp.sh

ENV CPLUS_INCLUDE_PATH="${CPLUS_INCLUDE_PATH}:/project/package/osqp/include"

# catkin build
RUN /bin/bash -c '. /opt/ros/noetic/setup.bash; catkin_make;'
```

## ROS 项目调试

项目构建成功后，可使用以下命令运行容器：

```bash
docker run -it --env="DISPLAY" --env="QT_X11_NO_MITSHM=1" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" --name=ros-motion-planning-noetic ros-motion-planning:noetic /bin/bash
```

利用vscode的插件`Remote Development`、`Docker`、`ROS`、`Remote Explorer`等插件，可以在vscode中直接连接到docker容器中，然后在vscode中打开项目文件夹，即可在vscode中进行ros项目开发调试。出于个人习惯，使用clang提供代码补全，可参见[这个博客](https://slam-learner.github.io/2024/04/01/%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE/clang/)进行设置。

创建`.vscode`文件夹，然后在其中创建`launch.json`、`tasks.json`、`c_cpp_properties.json`等文件。

`c_cpp_properties.json`文件如下：

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "/usr/include/**",
                "/usr/local/include/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/clang",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64",
            "compileCommands": "${workspaceFolder}/build/compile_commands.json",
            "configurationProvider": "ms-vscode.cmake-tools"
        }
    ],
    "version": 4
}
```

`tasks.json`文件如下：

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build_debug",
            "type": "shell",
            "command": "bash",
            "args": [
                "-c",
                "cd build_debug && cmake -DCMAKE_BUILD_TYPE=DEBUG .. && make -j4"
            ],
            "group": "build",
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared"
            }
        }
    ]
}
```

`launch.json`文件内容如下：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "ROS: launch",
            "type": "ros",
            "request": "launch",
            "target": "/project/src/sim_env/launch/main.launch",
            "preLaunchTask": "make_debug",
        }
    ]
}
```

点击运行，出现以下错误：

```bash
Error from roslaunch:
xacro: in-order processing became default in ROs Melodic. You can drop the option.
```

按下`ctrl+shift+f`搜索`--inorder`，将其删除，再次运行，可以解决此问题，但是又出现了以下错误：

```bash
Unable to start debugging.Launch options string provided by the project system is
invalid.Unable to determine path to debugger.Please specify the
"MIDebuggerPath"option.
```

根据ROS插件的一个[GitHub Issue](https://github.com/ms-iot/vscode-ros/issues/588)，这是由于ROS插件基于`gdb`进行调试，而此镜像没有安装`gdb`，使用`apt-get install gdb`安装`gdb`，再次运行，可以解决此问题，此时出现以下错误提示：

```bash
ROS launch can not load launch file that includes gazebo (program path 'PATH' is missing or invalid
```

根据[issue-474](https://github.com/ms-iot/vscode-ros/issues/474)以及插件的官方说明，插件无法调试`gazebo`，因此需要在`launch.json`文件中将`gazebo`相关的`launch`文件注释掉，再次运行，可以正常启动`ros`项目。
![20240410111451](https://raw.githubusercontent.com/slam-learner/Figures/main/PicGo/20240410111451.png)
根据插件中的说明，修改`launch.json`屏蔽`gazebo`等插件，内容如下：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "ROS: launch",
            "type": "ros",
            "request": "launch",
            "target": "/project/src/sim_env/launch/main.launch",
            "launch": ["rviz", "gz", "gzserver", "gzclient"],
            "preLaunchTask": "make_debug",
        }
    ]
}
```

在配置完全部环境之后，可以将配置好的环境重新打包成镜像，以便下次使用，具体的步骤如下：

```bash
# 1. check the container id
docker ps -a
# 2. commit the container
docker commit <container_id> ros-motion-planning:vscode-ros-dev
# 3. save the image
docker save -o ros-motion-planning-vscode-ros-dev.tar ros-motion-planning:vscode-ros-dev
```
