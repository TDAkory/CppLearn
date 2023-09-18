# SIMD Shuffle

## `_mm_shuffle_epi8`

```cpp
__m128i _mm_shuffle_epi8 (__m128i a, __m128i b)
```

Shuffle packed 8-bit integers in a according to shuffle control mask in the corresponding 8-bit element of b, and store the results in dst.

```shell
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=6097,6097&text=_mm_shuffle
FOR j := 0 to 15
	i := j*8
	IF b[i+7] == 1
		dst[i+7:i] := 0
	ELSE
		index[3:0] := b[i+3:i]
		dst[i+7:i] := a[index*8+7:index*8]
	FI
ENDFOR
```

Another way of understanding it:

> [Micorsoft Alphabetical Listing of Intrinsic Functions](https://learn.microsoft.com/zh-cn/previous-versions/visualstudio/visual-studio-2010/bb531427%28v=vs.100%29)

```cpp
// @note a A 128-bit parameter that contains sixteen 8-bit integers.
// @note mask A 128-bit byte mask.
__m128i _mm_shuffle_epi8( 
   __m128i a,
   __m128i mask
);

// The return value can be expressed by the following equations:
// r0 = (mask0 & 0x80) ? 0 : SELECT(a, mask0 & 0x0f)
// r1 = (mask1 & 0x80) ? 0 : SELECT(a, mask1 & 0x0f)
// ...
// r15 = (mask15 & 0x80) ? 0 : SELECT(a, mask15 & 0x0f)
```

r0-r15 and mask0-mask15 are the sequentially ordered 8-bit components of return value r and parameter mask. r0 and mask0 are the least significant 8 bits.

SELECT(a, n) extracts the nth 8-bit parameter from a. The 0th 8-bit parameter is the least significant 8-bits.

mask provides the mapping of bytes from parameter a to bytes in the result. If the byte in mask has its highest bit set, the corresponding byte in the result will be set to zero.
