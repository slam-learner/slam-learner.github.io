---
title: CMakeLists模板
date: 2024-02-06 17:09:04
tags: 
- CMake
categories:
- 模板
---

## CMakeLists模板

<!-- more -->

常用的项目文件`CMakeLists.txt`模板如下：

```cmake
cmake_minimum_required(VERSION 3.10)
project(PROJECT_NAME)

# ------------------- 编译选项 -------------------
IF(NOT CMAKE_BUILD_TYPE)
  SET(CMAKE_BUILD_TYPE RELEASE)
ENDIF()

if( CMAKE_BUILD_TYPE MATCHES DEBUG )
   message("Debug mode")
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}  -Wall -msse3 -std=c++11 -pthread -O0 -g -march=native -Wno-deprecated-declarations")
   set(TYPE debug)
else( CMAKE_BUILD_TYPE MATCHES RELEASE )
   message("Release mode")
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}  -Wall -msse3 -std=c++11 -pthread -O2 -march=native -Wno-deprecated-declarations -Wno-deprecated")
   set(TYPE release)
endif( CMAKE_BUILD_TYPE MATCHES DEBUG ) 

MESSAGE("Build type: " ${CMAKE_BUILD_TYPE})

set(CMAKE_EXPORT_COMPILECOMMANDS ON) 

# Check C++11 or C++0x support
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
   add_definitions(-DCOMPILEDWITHC11)
   message(STATUS "Using flag -std=c++11.")
elseif(COMPILER_SUPPORTS_CXX0X)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
   add_definitions(-DCOMPILEDWITHC0X)
   message(STATUS "Using flag -std=c++0x.")
else()
   message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

# ------------------- 依赖设置 -------------------
LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules)

find_package(YOUR_PKG REQUIRED)

# ------------------- 编译文件并设置路径 -------------------
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin/${TYPE})

aux_source_directory(${PROJECT_SOURCE_DIR}/src LIB_SRC)
add_library(${PROJECT_NAME}_lib SHARED
   ${LIB_SRC}
)

target_link_libraries(${PROJECT_NAME}_lib
   ${YOUR_PKG_LIBS}
)

# Build example
add_executable(BIN_NAME SOURCE_FILE.cc)
target_link_libraries(BIN_NAME ${PROJECT_NAME}_lib)
```

禁用内部构建`PreventInSourceBuild.cmake`：

```cmake
function(prevent_in_source_build)
    get_filename_component(src_dir "${CMAKE_SOURCE_DIR}" REALPATH)
    get_filename_component(bin_dir "${CMAKE_BINARY_DIR}" REALPATH)

    if ("${src_dir}" STREQUAL "${bin_dir}")
        message(FATAL_ERROR "In-source builds are not allowed.")
    endif()
endfunction(prevent_in_source_build)

message(STATUS "prevent_in_source_build.cmake loaded.")
prevent_in_source_build()
```

将项目内部安装库添加到路径中`SetupDependencies.cmake`：

```cmake
message(STATUS "Setting up dependencies...")

list(PREPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/../cmake/modules")
list(PREPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_LIST_DIR}/../install")
```

