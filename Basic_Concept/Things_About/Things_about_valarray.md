# [valarray](https://en.cppreference.com/w/cpp/numeric/valarray)

`std::valarray` is the class for representing and manipulating arrays of values. It supports element-wise mathematical operations and various forms of generalized subscript operators, slicing and indirect access.

```cpp
#include <valarray>
#include <vector>

std::valarray<int> get_data();

void use_valarray() {
    auto data = get_data();
    data += 4;
}

std::vector<int> get_vec_data();

void use_vector() {
    auto data = get_vec_data();

    for (auto &item: data) {
        item += 4;
    }
}
```