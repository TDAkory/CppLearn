# std::linalg: Linear Algebra Coming to Standard C++ - Mark Hoemmen - CppCon 2023

> https://www.youtube.com/watch?v=-UXHMlAMXNk&list=PLHTh1InhhwT7gQEuYznhhvAYTel0qzl72&index=2

## 线性代数库的责任

线性代数库在不同的抽象层级上承担着不同的责任和功能。

- Layer -1: 基础
  - **多维数组和迭代**：提供基本的数组操作和迭代功能。

- Layer 0: 性能原语
  - **向量操作**：
    - 向量点乘
    - 范数
    - 向量和
    - 平面旋转
  - **矩阵-向量操作**：
    - 矩阵-向量乘法
    - 三角矩阵求解
  - **矩阵-矩阵操作**：
    - 矩阵乘法
    - 带多个向量的三角矩阵求解
    - 对称矩阵更新

- Layer 1: 低级数学问题
  - **线性系统问题** \( Ax = b \)（包括行列式等）。
  - **最小二乘问题** \( \min ||Ax - b||^2 \)。
  - **特征值和奇异值问题** \( Ax = \lambda x \)。

- Layer 2: 高级数学问题
  - **统计推断**。
  - **物理模拟**。

## [std::mdspan](https://en.cppreference.com/w/cpp/container/mdspan)

> A C++23 feature



## std::linalg

> A C++26 feature

- [Basic linear algebra algorithms (since C++26)](https://en.cppreference.com/w/cpp/numeric/linalg)

```cpp
#include <cassert>
#include <cstddef>
#include <execution>
#include <linalg>
#include <mdspan>
#include <numeric>
#include <vector>
 
int main()
{
    constexpr std::size_t N = 40;
    std::vector<double> x_vec(N);
    std::ranges::iota(x_vec, 0);
 
    std::mdspan x(x_vec.data(), N);
 
    // x[i] *= 2.0, executed sequentially
    std::linalg::scale(2.0, x);
 
    // x[i] *= 3.0, executed in parallel
    std::linalg::scale(std::execution::par_unseq, 3.0, x);
 
    for (std::size_t i{}; i != N; ++i)
        assert(x[i] == 6.0 * static_cast<double>(i));
}
```
