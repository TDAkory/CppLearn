# Move

## `_mm_srli_epi16`

```cpp
__m128i _mm_srli_epi16 (__m128i a, int imm8)
```

Shift packed 16-bit integers in a right by imm8 while shifting in zeros, and store the results in dst.

```shell
FOR j := 0 to 7
	i := j*16
	IF imm8[7:0] > 15
		dst[i+15:i] := 0
	ELSE
		dst[i+15:i] := ZeroExtend16(a[i+15:i] >> imm8[7:0])
	FI
ENDFOR
```
see also `_mm_srli_epi32` `_mm_srli_epi64`.

## `_mm_movemask_epi8`

```cpp
int _mm_movemask_epi8 (__m128i a)
```

Create mask from the most significant bit of each 8-bit element in a, and store the result in dst.

```shell
FOR j := 0 to 15
	i := j*8
	dst[j] := a[i+7]
ENDFOR
dst[MAX:16] := 0
```
