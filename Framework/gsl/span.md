# [`std::span` / `gsl::span`](https://hackingcpp.com/cpp/std/span.html)

Replaces (Pointer, Length) Pairs

* lightweight  (= cheap to copy, can be passed by value)
* non-owning view  (= not responsible for allocating or deleting memory)
* of a contiguous memory block  (of e.g., std::vector, C-array, …)
* primary use case: as function parameter  (container-independent access to values)

```cpp
#include <span>  // C++20
#include <gsl/span> // C++14 + GSL

span<int>;	// sequence of integers whose values can be changed
span<int const>;	 // sequence of integers whose values can't be changed
span<int,5>;	// sequence of exactly 5 integers  (number of values fixed at compile time)
```