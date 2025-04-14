# SIMD Basic Knowledge

#SIMD

> [Writing C++ Wrappers for SIMD Intrinsics](https://johanmabille.github.io/blog/2014/10/09/writing-c-plus-plus-wrappers-for-simd-intrinsics-1/)

## 头文件

SSE/AVX指令主要定义于以下一些头文件中：

* `<xmmintrin.h>` : SSE, 支持同时对4个32位单精度浮点数的操作。
* `<emmintrin.h>` : SSE 2, 支持同时对2个64位双精度浮点数的操作。
* `<pmmintrin.h>` : SSE 3, 支持对SIMD寄存器的水平操作(horizontal operation)，如hadd, hsub等...。
* `<tmmintrin.h>` : SSSE 3, 增加了额外的instructions。
* `<smmintrin.h>` : SSE 4.1, 支持点乘以及更多的整形操作。
* `<nmmintrin.h>` : SSE 4.2, 增加了额外的instructions。
* `<immintrin.h>` : AVX, 支持同时操作8个单精度浮点数或4个双精度浮点数。

每一个头文件都包含了之前的所有头文件，所以如果你想要使用SSE4.2以及之前SSE3, SSE2, SSE中的所有函数就只需要包含<nmmintrin.h>头文件。

## 命名规则

* 数据类型通常以`_mxxx[T]`的方式进行命名。
  * 其中`xxx`代表数据的位数，如SSE提供的`__m128`为128位，AVX提供的`__m256`为256位。
  * `T`为类型，若为单精度浮点型则省略，若为整形则为`i`，如`__m128i`，若为双精度浮点型则为d，如`__m256d`。
* 操作浮点数的内置函数命名方式为：`_mm(xxx)_name_PT`。 
  * `xxx`为SIMD寄存器的位数，若为128则省略，如`_mm_addsub_ps`
  * `name`为函数执行的操作的名字，如加法为`_mm_add_ps`，减法为`_mm_sub_ps`
  * P代表的是对矢量(packed data vector)还是对标量(scalar)进行操作
  * T代表浮点数的类型，若为`s`则为单精度浮点型，若为`d`则为双精度浮点
* 操作整形的内置函数命名通常为：`_mm(XXX)_NAME_EPSYY`
  * `xxx`为SIMD寄存器的位数，若为128位则省略。
  * `name`为函数的名字。
  * `S`为整数的类型，若为无符号类型则为u，否在为i
  * `YY`为操作的数据类型的位数

## 函数类型

* 算术：`_mm_add_xx`, `_mm_sub_xx`, `_mm_mul_xx`, `_mm_div_xx`, …
* 逻辑：`_mm_and_xx`, `_mm_or_xx`, `_mm_xor_xx`, …
* 比较：`_mm_cmpeq_xx`, `_mm_cmpneq_xx`, `_mm_cmplt_xx`, …
* 转换：`_mm_cvtepixx`, …
* 内存拷贝：`_mm_load_xx`, `_mm_store_xx`, …
* 赋值：`_mm_set_xx`, `_mm_setzero_xx`, …
