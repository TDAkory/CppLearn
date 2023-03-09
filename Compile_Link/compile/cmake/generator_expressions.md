# [generator expressions](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html#manual:cmake-generator-expressions(7))

Generator expressions are evaluated during build system generation to produce information specific to each build configuration. They have the form $<...>. For example:

```shell
target_include_directories(tgt PRIVATE /opt/include/$<CXX_COMPILER_ID>)
```

This would expand to `/opt/include/GNU`, `/opt/include/Clang`, etc. depending on the C++ compiler used.