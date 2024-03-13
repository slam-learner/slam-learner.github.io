---
title: bash脚本
date: 2024-03-13 13:52:04
tags: 
- bash
categories:
- 模板
---

在项目本地编译安装第三方库而不是安装到系统目录

```bash
#!/bin/bash
##
## author: ydx
## description: 在项目本地编译安装第三方库而不是安装到系统目录
## date: 2023-12-25

# 记录脚本、项目目录
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" > /dev/null 2>&1 && pwd )"
PROJECT_DIR="$SCRIPT_DIR/.."

# 如果没有指定编译类型，则默认为Release
BUILD_TYPE="${1:-Release}"
CLEAN_BUILD="${2:-false}"
echo "Build type of third party: $BUILD_TYPE"

SPLIT_LINE="####################################################################"

# set -x # 打开调试模式，每行命令执行前都会打印该命令
set -e # 出错则退出

# 检查gcc和g++是否安装, 如果没有安装则安装gcc和g++
which gcc > /dev/null 2>&1 || { echo "gcc not found! Trying to install gcc!"; sudo apt-get install gcc; }
which g++ > /dev/null 2>&1 || { echo "g++ not found! Trying to install g++!"; sudo apt-get install g++; }

# 对于C++17的良好支持，需要g++9以上版本，这里进行检查
if [ -z "$CXX"]; then
    for VER in 9 10 11 12 13; do
        if hash g++-$VER 2> /dev/null; then
            export CXX=g++-$VER
            export CC=gcc-$VER
            break
        fi
    done
fi

# 如果没有找到g++9以上版本，则尝试更新gcc和g++到gcc-9 g++-9
# if [ -z STRING ] 测试STRING是否为空，如果为空返回 true
if [ -z "$CXX" ]; then
    echo "No g++ version 9 or higher found! Trying to update gcc and g++ to gcc-9 g++-9 for C++17 support!"
    sudo sh $SCRIPT_DIR/update-gcc-g++.sh
    if hash g++-9 2> dev/null; then
        export CXX=g++-9
        export CC=gcc-9
    fi
fi

# 检查gcc和g++编译器是否存在
# if [ -n STRING ] 测试STRING是否为空，如果非空返回 true
if [ -n "$CXX" ]; then
    which "$CXX"
else
    echo "No g++ version 9 or higher found! Please install g++-9 or higher!"
    exit 1
fi
if [ -n "$CC" ]; then
    which "$CC"
else
    echo "No gcc version 9 or higher found! Please install gcc-9 or higher!"
    exit 1
fi

# 使用同样的标准对各个库进行编译
YHS_CXX_STANDARD=14
echo "C++ standard: $YHS_CXX_STANDARD"

# 获取逻辑CPU核心数，用于并行编译，如果获取失败则默认为1
NUM_CORES=`(which nproc > /dev/null 2>&1 && nproc) || sysctl -n hw.logicalcpu || echo 1`
NUM_PARALLEL_BUILDS=$NUM_CORES
echo "Number of build thread: $NUM_CORES"

# Eigen默认使用16字节的对齐，但当AVX指令集可用时，Eigen会使用32字节的对齐
# 如果各个库或者项目使用了不同的对齐方式，可能会导致内存访问错误
# 而 Ceres 库以及 OpenGV 库在release模式会强制性传递 arch=native 选项
# 在现代的CPU上，arch=native 选项会启用 SSE4.2 和 AVX 指令集也就是会使用32字节的对齐
# 为了防止Eigen库引入一些微妙的bug，需要在各处编译时都加入arc=native选项
# 参见：https://eigen.tuxfamily.org/dox/TopicPreprocessorDirectives.html
if [ "$(uname -m)" = "x86_64" ]; then
    CMAKE_MARCH="{CXX_MARCH:-native}"
fi

if [ ! -z "$CXX_MARCH" ]; then
    EXTRA_CXX_FLAGS="$EXTRA_CXX_FLAGS -march=$CXX_MARCH"
fi

EXTRA_CXX_FLAGS="${EXTRA_CXX_FLAGS} ${CXX_FLAGS}"

INSTALL_DIR="$PROJECT_DIR/install"

# 通用的CMake参数
# -DCMAKE_BUILD_TYPE=${BUILD_TYPE} 指定编译类型
# -DCMAKE_CXX_STANDARD=${YHS_CXX_STANDARD} 指定C++标准
# -DCMAKE_CXX_FLAGS="${EXTRA_CXX_FLAGS}" 指定额外的C++编译选项
# -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} 指定安装目录
# -DCMAKE_PREFIX_PATH=${INSTALL_DIR} 指定查找依赖库的路径, find_package()会在这个路径下查找依赖库
# -DCMAKE_CXX_EXTENSIONS=OFF 禁用C++扩展
# -DCMAKE_POSITION_INDEPENDENT_CODE=ON 生成位置无关的代码
# -DBUILD_SHARED_LIBS=OFF 禁用动态库
COMMON_CMAKE_ARGS=(
    -DCMAKE_BUILD_TYPE=${BUILD_TYPE}
    -DCMAKE_CXX_STANDARD=${YHS_CXX_STANDARD}
    -DCMAKE_CXX_FLAGS="${EXTRA_CXX_FLAGS}"
    -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}
    -DCMAKE_PREFIX_PATH=${INSTALL_DIR}
    -DCMAKE_CXX_EXTENSIONS=OFF
    -DCMAKE_POSITION_INDEPENDENT_CODE=ON
    -DBUILD_SHARED_LIBS=OFF
)


# 检查ccache是否可用，如果可用则使用ccache加速编译
if command -v ccache > /dev/null 2>&1; then
    echo "ccache found! Using ccache to speed up compilation!"
    COMMON_CMAKE_ARGS+=(
            -DCMAKE_C_COMPILER_LAUNCHER=ccache
            -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
            "${COMMON_CMAKE_ARGS[@]}"
        )
else
    echo "ccache not found! Compiling without ccache!"
fi

rm -rf "$INSTALL_DIR"
mkdir -p "$INSTALL_DIR"

# 由于有的库之间存在依赖关系，并且有的库需要额外添加一些编译选项，所以需要按照顺序手工编译
THIRD_DIR="$PROJECT_DIR/third_party"
##################################
## Eigen
if true; then
    echo ""
    echo "$SPLIT_LINE"
    echo "$SPLIT_LINE"
    echo ""

    echo "Building Eigen..."
    cd "$THIRD_DIR/eigen-3.4.0"
    if [ "$CLEAN_BUILD" = true ]; then
        rm -rf build
    fi
    mkdir -p build
    cd build
    cmake .. "${COMMON_CMAKE_ARGS[@]}" \
        -DBUILD_TESTING=OFF
    make -j$NUM_PARALLEL_BUILDS
    make install
fi

##################################
## yaml-cpp
if true; then
    echo ""
    echo "$SPLIT_LINE"
    echo "$SPLIT_LINE"
    echo ""

    echo "Building yaml-cpp..."
    cd "$THIRD_DIR/yaml-cpp"
    if [ "$CLEAN_BUILD" = true ]; then
        rm -rf build
    fi
    mkdir -p build
    cd build
    cmake .. "${COMMON_CMAKE_ARGS[@]}"
    make -j$NUM_PARALLEL_BUILDS
    make install
fi

##################################
## OpenCV 4.5.4
if true; then
    echo ""
    echo "$SPLIT_LINE"
    echo "$SPLIT_LINE"
    echo ""

    echo "Building OpenCV..."
    cd "$THIRD_DIR/opencv-4.5.4"
    if [ "$CLEAN_BUILD" = true ]; then
        rm -rf build
    fi
    mkdir -p build
    cd build
    cmake .. "${COMMON_CMAKE_ARGS[@]}" \
        -DCMAKE_VERBOSE_MAKEFILE=ON \
        -DOPENCV_ENABLE_NONFREE=ON \
        -DOPENCV_GENERATE_PKGCONFIG=ON \
        -DWITH_IPP=ON -DWITH_TBB=ON -DWITH_OPENMP=ON -DWITH_PTHREADS_PF=ON \
        -DWITH_1394=OFF \
        -DBUILD_TESTS=OFF \
        -DBUILD_PERF_TESTS=OFF \
        -DENABLE_CXX11=ON \
        -DBUILD_DOCS=OFF \
        -DBUILD_EXAMPLES=OFF \
        -DBUILD_JASPER=OFF \
        -DBUILD_OPENEXR=OFF \
        -DWITH_FFMPEG=ON \
        -DWITH_EIGEN=ON \
        -DWITH_LAPACK=OFF
    make -j12
    make install
fi

##################################
## gtsam
if true; then
    echo ""
    echo "$SPLIT_LINE"
    echo "$SPLIT_LINE"
    echo ""

    echo "Building gtsam..."
    cd "$THIRD_DIR/gtsam-4.0.3"
    if [ "$CLEAN_BUILD" = true ]; then
        rm -rf build
    fi
    mkdir -p build
    cd build
    cmake .. "${COMMON_CMAKE_ARGS[@]}"
    make -j$NUM_PARALLEL_BUILDS
    make install
fi

echo ""
echo "$SPLIT_LINE"
echo "$SPLIT_LINE"
echo ""
echo "All third party libraries are built and installed to $INSTALL_DIR!"
```

更新gcc和g++到gcc-9 g++-9

```bash
#!/usr/bin/env bash
##
## author: ydx
## description: update gcc and g++ to gcc-9 g++-9 for C++17 support
## date: 2023-12-25

echo "Trying to update gcc and g++ to gcc-9 g++-9 for C++17 support!"
echo "You may need to enter your password!"

echo "add-apt-repository ppa:ubuntu-toolchain-r/test"
sudo add-apt-repository ppa:ubuntu-toolchain-r/test

echo "update apt-get"
sudo apt-get update

echo "install gcc-9 g++-9, wait a moment!"
sudo apt-get install gcc-9 g++-9

echo "set gcc-9 g++-9 as default gcc g++"
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 20 --slave /usr/bin/g++ g++ /usr/bin/g++-9
```

更新cmake到3.22.1

```bash
#!/usr/bin/env bash
##
## author: ydx
## description: 升级cmake到3.22.1
## date: 2023-12-25

set -e

CMAKE_CURRENT_VERSION=`cmake --version | grep version | awk '{print $3}'`

echo "Trying to update cmake"

cd ~/package

echo "Downloading cmake-3.22.1.tar.gz"
wget https://cmake.org/files/v3.22/cmake-3.22.1.tar.gz

echo "Unpacking cmake-3.22.1.tar.gz"
tar -zxvf cmake-3.22.1.tar.gz

echo "Installing cmake-3.22.1"
cd cmake-3.22.1
chmod a+x ./configure
sh ./configure

NUM_CORES=`(which nproc > /dev/null 2>&1 && nproc) || sysctl -n hw.logicalcpu || echo 1`
make -j$NUM_CORES

echo "Trying to install cmake-3.22.1 to system directory, you may need to enter your password!"
sudo make install
sudo update-alternatives --install /usr/bin/cmake cmake /usr/local/bin/cmake 1 --force

CMAKE_NEW_VERSION=`cmake --version | grep version | awk '{print $3}'`

if [ "$CMAKE_NEW_VERSION" = "$CMAKE_CURRENT_VERSION" ]; then
    echo "Failed to update cmake!"
    exit 1
else
    echo "Successfully update cmake from $CMAKE_CURRENT_VERSION to $CMAKE_NEW_VERSION!"
fi
```
