# How to use cmake

## 目录

## 基本用法

## 标识符

### cmake_minimum_required

```c
cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
```

隐式调用cmake_policy(VERSION 3.5)，用于设置工程最低适用的CMake版本及构建规则，如低于要求则停止构建并报错。

* VERSION 3.5规定了最低的CMake适用版本，如果实际版本高于3.5，仍以3.5作为构建规则
* FATAL_ERROR迫使CMake≤2.4时，如不满足版本要求则停止构建并报错，而不仅仅是警告，如CMake≥2.6则忽略
* 该命令设置于工程最上级CMakeLists.txt
* 推荐使用3.1+的版本

