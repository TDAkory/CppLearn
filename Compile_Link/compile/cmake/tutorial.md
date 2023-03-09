# [Cmake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)

- [Cmake Tutorial](#cmake-tutorial)
  - [A Basic Starting Point](#a-basic-starting-point)
    - [Building a Basic Project](#building-a-basic-project)
    - [Specifying the C++ Standard](#specifying-the-c-standard)
    - [Adding a Version Number and Configured Header File](#adding-a-version-number-and-configured-header-file)
  - [Adding a Library](#adding-a-library)
  - [Adding Usage Requirements for a Library](#adding-usage-requirements-for-a-library)
    - [Adding Usage Requirements for a Library](#adding-usage-requirements-for-a-library-1)
  - [Adding Generator Expressions](#adding-generator-expressions)
    - [Setting the C++ Standard with Interface Libraries](#setting-the-c-standard-with-interface-libraries)
    - [Adding Compiler Warning Flags with Generator Expressions](#adding-compiler-warning-flags-with-generator-expressions)
  - [Installing and Testing](#installing-and-testing)
  - [Adding Support for a Testing Dashboard](#adding-support-for-a-testing-dashboard)
  - [Adding System Introspection](#adding-system-introspection)


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

## Adding Usage Requirements for a Library

### Adding Usage Requirements for a Library

Usage requirements of a target parameters allow for far better control over a library or executable's link and include line while also giving more control over the transitive property of targets inside CMake. The primary commands that leverage usage requirements are:

* target_compile_definitions()
* target_compile_options()
* target_include_directories()
* target_link_directories()
* target_link_options()
* target_precompile_headers()
* target_sources()

## Adding Generator Expressions

### Setting the C++ Standard with Interface Libraries

Add an INTERFACE library target to specify the required C++ standard.

create an interface library, tutorial_compiler_flags.

use `target_compile_features()` to add the compiler feature cxx_std_11.

```shell
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)

target_link_libraries(Tutorial PUBLIC ${EXTRA_LIBS} tutorial_compiler_flags)
```

### Adding Compiler Warning Flags with Generator Expressions

A common usage of `generator expressions` is to conditionally add compiler flags, such as those for language levels or warnings. A nice pattern is to associate this information to an INTERFACE target allowing this information to propagate.

determine which compiler our system is currently using to build since warning flags vary based on the compiler we use. This is done with the COMPILE_LANG_AND_ID generator expression. We set the result in the variables gcc_like_cxx and msvc_cxx as follows:

```shell
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
```

We use target_compile_options() to apply these flags to our interface library. we only want these warning flags to be used during builds. Consumers of our installed project should not inherit our warning flags. To specify this, we wrap our flags in a generator expression using the `BUILD_INTERFACE` condition.

```shell
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
  "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)
```

## Installing and Testing

normally we have bin for executable, include for header files, and lib for libraries.

```shell
set(installable_libs MathFunctions tutorial_compiler_flags)
install(TARGETS ${installable_libs} DESTINATION lib)

install(FILES MathFunctions.h DESTINATION include)

install(TARGETS Tutorial DESTINATION bin)
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"
  DESTINATION include
  )
```

CTest offers a way to easily manage tests for your project. Tests can be added through the `add_test()` command. Although it is not explicitly covered in this tutorial, there is a lot of compatibility between CTest and other testing frameworks such as GoogleTest.

## [Adding Support for a Testing Dashboard](https://cmake.org/cmake/help/latest/guide/tutorial/Adding%20Support%20for%20a%20Testing%20Dashboard.html)

> Not that important

## Adding System Introspection

1. Include the CheckCXXSourceCompiles module.
2. use check_cxx_source_compiles to determine whether log and exp are available from cmath.

```shell
include(CheckCXXSourceCompiles)

check_cxx_source_compiles("
  #include <cmath>
  int main() {
    std::log(1.0);
    return 0;
  }
" HAVE_LOG)
check_cxx_source_compiles("
  #include <cmath>
  int main() {
    std::exp(1.0);
    return 0;
  }
" HAVE_EXP)

if (HAVE_LOG AND HAVE_EXP)
    target_compile_definitions(MathFunctions PRIVATE "HAVE_LOG" "HAVE_EXP")
endif()
```
