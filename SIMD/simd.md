# [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data)

## Advantages

multimedia & DSP

## Disadvantages

* a flow-control-heavy task like code parsing may not easily benefit from SIMD
* Currently, implementing an algorithm with SIMD instructions usually requires human labor; most compilers don't generate SIMD instructions from a typical C program
* Programming with particular SIMD instruction sets can involve numerous low-level challenges.

## Hardware

Intel's MMX and iwMMXt, SSE, SSE2, SSE3 SSSE3 and SSE4.x, AMD's 3DNow!, ARM's Neon technology.
Intel's AVX-512 SIMD instructions process 512 bits of data at once.

## SMID on C++

### 📝 常用SIMD指令集头文件与校验宏

| 架构 | SIMD指令集 | 头文件 | 校验宏 |
| :--- | :--- | :--- | :--- |
| **x86** | **SSE** | `xmmintrin.h` | `__SSE__` |
| | **SSE2** | `emmintrin.h` | `__SSE2__` |
| | **SSE3** | `pmmintrin.h` | `__SSE3__` |
| | **SSSE3** | `tmmintrin.h` | `__SSSE3__` |
| | **SSE4.1** | `smmintrin.h` | `__SSE4_1__` |
| | **SSE4.2** | `nmmintrin.h` | `__SSE4_2__` |
| | **AES, PCLMUL** | `wmmintrin.h` | `__AES__`, `__PCLMUL__` |
| | **AVX** | `avxintrin.h` | `__AVX__` |
| | **AVX2** | `avx2intrin.h` | `__AVX2__` |
| | **FMA** | `fmaintrin.h` | `__FMA__` |
| | **综合头文件** | `immintrin.h` (常用) | 取决于启用的子集 |
| | | `x86intrin.h` (通用) | 取决于启用的子集 |
| **ARM** | **NEON** (32/64-bit) | `arm_neon.h` | `__ARM_NEON__` 或 `__ARM_NEON` |
| | **SVE** (可伸缩矢量) | `arm_sve.h` | `__ARM_FEATURE_SVE` |

**平台与编译器检测**：在包含特定架构的头文件前，务必先检测当前编译的架构和编译器。一个常见的跨平台检测模式如下：

```c
#if defined(__GNUC__) && (defined(__x86_64__) || defined(__i386__))
    // GCC兼容编译器，目标为x86/x86-64
    #include <x86intrin.h>
#elif defined(__GNUC__) && defined(__ARM_NEON__)
    // GCC兼容编译器，目标为支持NEON的ARM
    #include <arm_neon.h>
#endif
```

`__x86_64__`、`__i386__`、`__arm__`、`__aarch64__` 是判断平台的基础宏。

1.  **校验宏的生效条件**：表格中列出的校验宏（如 `__SSE2__`、`__ARM_NEON__`）通常**只有在编译器启用了相应指令集选项时才会被定义**。例如，GCC/Clang中，编译时使用 `-msse2` 选项，`__SSE2__` 宏才会被定义。

2.  **兼容性问题**：不应无条件包含特定架构的头文件。例如，在ARM设备上若编译器未启用NEON支持，直接包含 `arm_neon.h` 可能导致编译失败。正确的做法是在包含前检查 `__ARM_NEON__` 宏。

3.  **编译器参数**：要在代码中使用特定SIMD指令集，除了包含头文件，还需在编译时开启对应支持。例如：
    - x86: `-msse`, `-msse2`, `-mavx`, `-mavx2`
    - ARM: `-mfpu=neon` (armv7), `-march=armv8-a+simd` (aarch64)

4.  **跨平台代码移植**：如果你需要将x86的SIMD代码迁移到ARM平台，可能需要寻找功能等效的指令进行重写。社区有一些开源项目可以辅助这项工作，例如：
    - **sse2neon**：将x86 SSE intrinsics代码转换为ARM NEON代码。
    - **SIMDe**：一个更全面的库，提供了一套跨平台的SIMD抽象，支持多种架构。

## ReadList

- [intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=3730,5200,1884,4635,466&techs=SSE_ALL,AVX_ALL,AVX_512)

## Use Case

- [SIMDized check which bytes are in a set](http://0x80.pl/articles/simd-byte-lookup.html)