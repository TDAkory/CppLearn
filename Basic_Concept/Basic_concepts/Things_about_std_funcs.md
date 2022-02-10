# Things about std funcs

## std::sort

* [sort里的comp需要是小于而不是小于等于，而且会用 `!comp(a, b) && !comp(b, a)` 判断相等](https://zh.cppreference.com/w/cpp/named_req/Compare)
* [std::sort with equal elements gives Segmentation fault](https://stackoverflow.com/questions/16535882/stdsort-with-equal-elements-gives-segmentation-fault)