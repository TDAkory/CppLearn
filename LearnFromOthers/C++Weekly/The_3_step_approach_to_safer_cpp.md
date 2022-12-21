# [Link](https://www.youtube.com/watch?v=dSYFm65KcYo)

## Step 0

* You already have tests
* And Continuous intergration/build/test

## Step 1 - Add Basic Static Analysis

1. Enable warning as errors
2. Fix all of the existing warnings
3. Build both `Release` and `Debug`
4. Add clang-tidy (disable project specific options (llvm,fuschia,google,tec...))
5. Increase warning level(-Wall to discover new warnings)
6. GOTO 2.

## Step 2 - Enable Sanitizers For Testing

* Address Sanitizer, UB Sanitizer, Thread Sanitizer, Memory ?
* `valgrind`, `Dr Memory`

## Step 3 - Add Fuzzing

* Finds the things you don't think about!
* All user facing APIs should be Fuzzed
