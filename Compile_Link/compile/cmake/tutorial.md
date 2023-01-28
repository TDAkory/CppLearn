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

To add a library in CMake, use the `add_library()` command and specify which source files should make up the library.

Rather than placing all of the source files in one directory, we can organize our project with one or more subdirectories. In this case, we will create a subdirectory specifically for our library. Here, we can add a new CMakeLists.txt file and one or more source files. In the top level CMakeLists.txt file, we will use the add_subdirectory() command to add the subdirectory to the build.

Once the library is created, it is connected to our executable target with target_include_directories() and target_link_libraries().
