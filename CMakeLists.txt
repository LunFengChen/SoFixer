cmake_minimum_required(VERSION 3.10...3.28)
project(SoFixer)

# 设置 C++ 标准的现代方式
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# =========================================================
# SoFixer options
# =========================================================
set(SO_64 OFF CACHE BOOL "build SoFixer for 64bit target")

if(SO_64)
    message("building SoFixer for 64bit target")
    add_definitions("-D__SO64__")
    set(TARGET_NAME SoFixer64)
else()
    message("building SoFixer for 32bit target")
    add_definitions("-D__SO32__")
    set(TARGET_NAME SoFixer32)
endif()



# Debug 模式添加调试符号
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0")
    message("Building in Debug mode with symbols")
endif()

if(MINGW)
    message("build with mingw")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static -static-libgcc -static-libstdc++")
endif()

set(ROOT_SRC ElfReader.cpp
        ElfRebuilder.cpp
        ObElfReader.cpp)

add_executable(${TARGET_NAME} ${ROOT_SRC} main.cpp)
