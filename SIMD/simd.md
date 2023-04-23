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

