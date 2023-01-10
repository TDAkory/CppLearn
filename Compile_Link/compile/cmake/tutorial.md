# [Cmake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)

## A Basic Starting Point

### Building a Basic Project

* Any project's top most CMakeLists.txt must start by specifying a minimum CMake version using the `cmake_minimum_required()` command. 
* use the `project()` command to set the project name. This call is required with every project and should be called soon after `cmake_minimum_required()`. 
* the `add_executable()` command tells CMake to create an executable using the specified source code files.

### Specifying the C++ Standard

* Two of these special user settable variables are `CMAKE_CXX_STANDARD` and `CMAKE_CXX_STANDARD_REQUIRED`. These may be used together to specify the C++ standard needed to build the project.

```CMakeLists.txt
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
```

### Adding a Version Number and Configured Header File

* One way to accomplish this is by using a configured header file. We create an input file with one or more variables to replace. These variables have special syntax which looks like `@VAR@`. Then, we use the `configure_file()` command to copy the input file to a given output file and replace these variables with the current value of VAR in the CMakelists.txt file.

## Adding a Library