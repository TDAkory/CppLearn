# And Or Not

## `_mm_and_si128`

```cpp
#include <emmintrin.h>

__m128i _mm_and_si128 (__m128i a, __m128i b)

// Instruction: pand xmm, xmm
// CPUID Flags: SSE2
```

Compute the bitwise AND of 128 bits (representing integer data) in a and b, and store the result in dst.

```shell
dst[127:0] := (a[127:0] AND b[127:0])
```
