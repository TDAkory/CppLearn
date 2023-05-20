# Protobuf with cmake

* [Protobuf在Cmake中的正确使用](https://oldpan.me/archives/protobuf-cmake-right-usage)
* [FindProtobuf](https://cmake.org/cmake/help/latest/module/FindProtobuf.html)

```cpp
find_package(Protobuf REQUIRED)
include_directories(${Protobuf_INCLUDE_DIRS})
include_directories(${CMAKE_CURRENT_BINARY_DIR})
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS foo.proto)
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS EXPORT_MACRO DLL_EXPORT foo.proto)
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS DESCRIPTORS PROTO_DESCS foo.proto)
protobuf_generate_python(PROTO_PY foo.proto)
add_executable(bar bar.cc ${PROTO_SRCS} ${PROTO_HDRS})
target_link_libraries(bar ${Protobuf_LIBRARIES})
```

***Note** The protobuf_generate_cpp and protobuf_generate_python functions and add_executable() or add_library() calls only work properly within the same directory.*