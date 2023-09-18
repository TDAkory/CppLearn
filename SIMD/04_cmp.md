# Cmp

## `_mm_cmpeq_epi8`

```cpp
__m128i _mm_cmpeq_epi8 (__m128i a, __m128i b)
```

Compare packed 8-bit integers in a and b for equality, and store the results in dst.

```shell
FOR j := 0 to 15
	i := j*8
	dst[i+7:i] := ( a[i+7:i] == b[i+7:i] ) ? 0xFF : 0
ENDFOR
```
