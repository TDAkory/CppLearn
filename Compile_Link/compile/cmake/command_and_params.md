# commands and parameters of CMake

- [commands and parameters of CMake](#commands-and-parameters-of-cmake)
  - [`PROJECT_SOURCE_DIR`](#project_source_dir)
  - [`CMAKE_CURRENT_SOURCE_DIR`](#cmake_current_source_dir)
  - [`PUBLIC` `PRIVATE` `INTERFACE`](#public-private-interface)
  - [`add_library`](#add_library)
  - [`option()`](#option)
  - [`install()`](#install)
  - [`CheckCXXSourceCompiles`](#checkcxxsourcecompiles)
  - [`find_package()`](#find_package)


## `PROJECT_SOURCE_DIR`

This is the source directory of the last call to the `project()` command made in the current directory scope or one of its parents. Note, it is not affected by calls to `project()` made within a child directory scope (i.e. from within a call to `add_subdirectory()` from the current scope).

## `CMAKE_CURRENT_SOURCE_DIR`

The path to the source directory currently being processed.
This is the full path to the source directory that is currently being processed by cmake.
When run in cmake -P script mode, CMake sets the variables CMAKE_BINARY_DIR, CMAKE_SOURCE_DIR, CMAKE_CURRENT_BINARY_DIR and CMAKE_CURRENT_SOURCE_DIR to the current working directory.

## `PUBLIC` `PRIVATE` `INTERFACE`

- [CMAKE 里PRIVATE、PUBLIC、INTERFACE属性示例详解](https://blog.csdn.net/weixin_43862847/article/details/119762230)
- [cmake：target_** 中的 PUBLIC，PRIVATE，INTERFACE](https://zhuanlan.zhihu.com/p/82244559)

## `add_library`

```makefile
add_library(<name> [STATIC | SHARED | MODULE]
            [EXCLUDE_FROM_ALL]
            [<source>...])
```

Adds a library target called `<name>` to be built from the source files listed in the command invocation. The `<name>` corresponds to the logical target name and must be globally unique within a project. The actual file name of the library built is constructed based on conventions of the native platform (such as `lib<name>.a` or `<name>.lib`).

## `option()`

```makefile
option(<variable> "<help_text>" [value])
```

If no initial <value> is provided, boolean OFF is the default value. If <variable> is already set as a normal or cache variable, then the command does nothing

## [`install()`](https://cmake.org/cmake/help/latest/command/install.html#command:install)

## [`CheckCXXSourceCompiles`](https://cmake.org/cmake/help/latest/module/CheckCXXSourceCompiles.html#module:CheckCXXSourceCompiles)

Check if given C++ source compiles and links into an executable.

## [`find_package()`](https://cmake.org/cmake/help/latest/command/find_package.html)

- [深入理解find_package用法](https://zhuanlan.zhihu.com/p/97369704)