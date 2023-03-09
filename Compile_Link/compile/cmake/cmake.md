# How to use [cmake](https://cmake.org/)

> CMake is an open-source, cross-platform family of tools designed to build, test and package software. CMake is used to control the software compilation process using simple platform and compiler independent configuration files, and generate native makefiles and workspaces that can be used in the compiler environment of your choice. The suite of CMake tools were created by Kitware in response to the need for a powerful, cross-platform build environment for open-source projects such as ITK and VTK

- [Mastering cmake](https://cmake.org/cmake/help/book/mastering-cmake/)

## 目录

## 基本用法

## 标识符

### `cmake_minimum_required`

```c
cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
```

隐式调用cmake_policy(VERSION 3.5)，用于设置工程最低适用的CMake版本及构建规则，如低于要求则停止构建并报错。

* VERSION 3.5规定了最低的CMake适用版本，如果实际版本高于3.5，仍以3.5作为构建规则
* FATAL_ERROR迫使CMake≤2.4时，如不满足版本要求则停止构建并报错，而不仅仅是警告，如CMake≥2.6则忽略
* 该命令设置于工程最上级CMakeLists.txt
* 推荐使用3.1+的版本

