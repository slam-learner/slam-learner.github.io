---
title: clang
date: 2024-04-01 13:40:07
tags:
  - clang
categories:
  - 环境配置
---

[参考资料 1]( https://zhuanlan.zhihu.com/p/551978555 )
[参考资料 2](https://zhuanlan.zhihu.com/p/445300679)

## 介绍

### LSP

如果想要了解clang 首先要明白 LSP（Language Server Protocol）：语言服务协议。

传统的 IDE，即使离线使用时也能完成语法检查、自动补全、跳转位置、引用、查找等功能。因为这些 IDE 的语法特性检查功能都是在本地实现的。不仅如此，各家 IDE 都有各家的实现。比如以 Java IDE 为例，对于 Java 的语法特性检查，IntelliJ IDEA 有其自己的实现方式，Eclipse 也有其自己的实现方式。这就造成了对于同一种编程语言的语法解析需要针对不同的 IDE 进行不同的适配。其次，语言扫描相关的工作都比较占用 CPU 资源，运行在 vscode 进程中不如单独放在一个独立进程，甚至远程服务器上更好。

由于各种新技术新语言层出不穷，VSCode 开发人员不可能对任何一种编程语言的语法都了如指掌，专业领域的编程语言的语法解析，由编程语言的开发者来实现是最好的。

LSP(Language Server Protocol) 语言服务协议，此协议定义了在编辑器与语言服务器之间使用的协议。

通过LSP使得编程语言社区能专注于不断完善一个能提供语法检查、自动补全、跳转位置、引用查找等语言特性检查的高性能 “语言服务器” 实现。与此同时，IDE厂商和组织只专注于提供能与任何 “语言服务器” 交互和 “渲染” 响应的实现方案。

### LLVM 与 Clang

简单来说，LLVM 是编译器后端，Clang 是编译器前端。

#### LLVM

LLVM（Low Level Virtual Machine）。LLVM 是以 BSD 许可来开发的开源的编译器框架系统，基于 C++ 编写而成，利用虚拟技术来优化以任意程序语言编写的程序的编译时间、链接时间、运行时间以及空闲时间，最早以 C/C++ 为实现对象，对开发者保持开放，并兼容已有脚本。

目前 LLVM 因其宽松的许可协议，更好的模块化、更清晰的架构，成为很多厂商或者组织的选择，已经被苹果 IOS 开发工具、Facebook、Google 等各大公司采用，像 Swift、Rust 等语言都选择了以 LLVM 为后端。

大多数编译器由两部分组成：前端和后端。

- 前端负责语法分析，生成中间代码；
- 后端以中间代码作为输入，进行架构无关的代码优化，接着针对不同架构生成不同的机器码。

这种架构使得前后端依赖统一格式的中间代码(IR)，使得前后端可以独立的变化。新增一门语言只需要修改前端，而新增一个CPU架构只需要修改后端即可。

  Objective C/C/C++使用的编译器前端是Clang，Swift使用的是Swift，后端都是LLVM。
  
#### Clang

在 LibTooling 的基础之上有个开发人员工具合集Clang tools。Clang Tools是为 C++ 开发人员设计的独立命令行（可能还有GUI）工具。目前只有几个最基础和最根本的的工具保存在主 Clang目录树中，其余的工具保存在一个单独的目录树中称为Extra Clang Tools；

Clang的Extra Clang Tools中有一个工具是**Clangd**。它是对LSP协议的一个具体实现（当然是建立在Clang的基础之上的），目的是为了**给C/C++的编辑器提供编程语言的一些智能化的特性**，比如代码分析、引用查找等。

  **LLDB**（Low-Level Debugger）则是一种开源的**调试器**，最初是作为 LLVM 项目的一部分而开发的。它设计用于支持多种编程语言，包括 C, C++, Objective-C, 和 Swift。

## Clang 的安装

[参考博客](https://blog.csdn.net/sinat_34986308/article/details/116116780)
在 Ubuntu18.04 中安装 Clang-12 以上版本的话需要新增软件源，按照以下步骤进行：

1.获取key

```bash
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
```

2.在 `/etc/apt/sources.list` 中添加下列文本:（如果不是18.04，需要把bionic替换为对应的版本，可以通过 `lsb_release -a` 查看）

```text
deb http://apt.llvm.org/bionic/ llvm-toolchain-bionic main
deb-src http://apt.llvm.org/bionic/ llvm-toolchain-bionic main
# 11
deb http://apt.llvm.org/bionic/ llvm-toolchain-bionic-11 main
deb-src http://apt.llvm.org/bionic/ llvm-toolchain-bionic-11 main
# 12
deb http://apt.llvm.org/bionic/ llvm-toolchain-bionic-12 main
deb-src http://apt.llvm.org/bionic/ llvm-toolchain-bionic-12 main
```

3.安装相关软件包

```bash
sudo apt update
sudo apt install clang-12 clangd-12 llvm-12 llvm-12-dev liblldb-12 liblldb-12-dev clang-format-12 bear clang-tidy-12
```

4.使用 update-alternative 来设置设置默认命令使用 12 版本提供的

```bash
sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-12 100
sudo update-alternatives --install /usr/bin/clangd clangd /usr/bin/clangd-12 100
sudo update-alternatives --install /usr/bin/lldb lldb /usr/bin/lldb-12 100
```

### 在 Vs Code 中的配置

必装： 1. clangd 2. CodeLLDB

选装： 3. CMake 4. CMake Tools 5. Clang-Format 6.  Code Runner

#### 在用户 json 中的配置

```json
{
    // code runner
    "code-runner.runInTerminal": true,
    "code-runner.saveFileBeforeRun": true, // run code前保存
    "code-runner.clearPreviousOutput": true, // 每次run code前清空属于code runner的终端消息，默认false
    "clangd.arguments": [
        "--compile-commands-dir=${workspaceFolder}/build",//指定配置文件compelie_commands.json所在目录，这里有三种方法生成
        // 在后台自动分析文件（基于complie_commands)
        "--background-index",
        // 同时开启的任务数量
        "-j=12",
        // "--folding-ranges"
        // 告诉clangd用那个clang进行编译，路径参考which clang++的路径
        "--query-driver=/usr/bin/clang++",
        // clang-tidy功能
        "--clang-tidy",
        "--clang-tidy-checks=performance-*,bugprone-*",
        // 全局补全（会自动补充头文件）
        "--all-scopes-completion",
        // 更详细的补全内容
        "--completion-style=detailed",
        "--function-arg-placeholders",
        // 补充头文件的形式
        "--header-insertion=iwyu",
        // pch优化的位置
        "--pch-storage=memory",
    ],
}
```

生成 clangd 配置文件

1.在 CMakeLists.txt 文件中添加 set(CMAKE_EXPORT_COMPILECOMMANDS ON) 运行

```bash
mkdir -p build
cd build && cmake ..
```

可以发现在 build目录下已经生成了 compile_commands.json文件

2.make 或其他项目

```bash
sudo apt install bear
bear make
```

#### launch. json

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "DomiMonoDebug",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/your-exe-path", // 可执行文件路径
            "args": [
                "arg1",
                "arg2"
            ], // 命令行参数
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "build_debug" // 预构建任务名称
        }
    ]
}
```

#### tasks. json

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

#### c_cpp_properties. json

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "${workspaceFolder}/include/**",
                "/usr/include/**",
                "/usr/local/include/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/clang",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "gcc-x64",
            "compileCommands": "${workspaceFolder}/build/compile_commands.json",
            "configurationProvider": "ms-vscode.cmake-tools"
        }
    ],
    "version": 4
}
```

有一篇[知乎文章]( https://zhuanlan.zhihu.com/p/579294471 ) 中提到：
>lldb 不自带 lldb-mi，但在 VSCode 进行调试时需要，所以要自己生成一下

需要进行以下操作：

```bash
sudo apt-get install liblldb-dev
cd ~
git clone https://github.com/lldb-tools/lldb-mi.git
cd lldb-mi
cmake .
cmake --build .
sudo cp src/lldb-mi /usr/bin/
```

其中编译会报错，按照提示，找到报错位置，注释掉相应的代码即可，但是可能不需要这个步骤也可以成功完成调试。
