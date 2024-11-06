# Use vcpkg with cmake

## 直接用，全局的方式

1. 安装vcpkg

```shell
git clone https://github.com/microsoft/vcpkg "$HOME/vcpkg"
cd vcpkg
./bootstrap-vcpkg.sh
```

2. 获取需要配置的路径名

```shell
./vcpkg integrate install
Applied user-wide integration for this vcpkg root.
CMake projects should use: "-DCMAKE_TOOLCHAIN_FILE=<root to vcpkg>/scripts/buildsystems/vcpkg.cmake"
```

3. 在构建中增加上述参数

4. 可以选择在参数中增加，也可以选择在CMakeLists.txt中增加

```cmake
set(CMAKE_TOOLCHAIN_FILE "<root to vcpkg>/scripts/buildsystems/vcpkg.cmake")
```