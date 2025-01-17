# [`std::set_difference`](https://en.cppreference.com/w/cpp/algorithm/set_difference)

## demo

```cpp
// Type your code here, or load an example.
#include <vector>
#include <set>
#include <unordered_set>
#include <iostream>
#include <algorithm>

using namespace std;

template<typename T>
void print(const T& set)
{
    for (const auto& elem : set)
        std::cout << elem << ' ';
    std::cout << '\n';
}

int main() {
    {
        unordered_set<int> a{4,5,5,7,8};
        unordered_set<int> b{2,4,5,6};
        print(a);
        print(b);

        unordered_set<int> upper;
        set_difference(a.begin(), a.end(), b.begin(), b.end(), std::inserter(upper, upper.end()));
        print(upper);

        unordered_set<int> lower;
        set_difference(b.begin(), b.end(), a.begin(), a.end(), std::inserter(lower, lower.end()));
        print(lower);
    }
    std::cout << '\n';
    {
        set<int> a{4,5,5,7,8};
        set<int> b{2,4,5,6};
        print(a);
        print(b);

        set<int> upper;
        set_difference(a.begin(), a.end(), b.begin(), b.end(), std::inserter(upper, upper.end()));
        print(upper);

        set<int> lower;
        set_difference(b.begin(), b.end(), a.begin(), a.end(), std::inserter(lower, lower.end()));
        print(lower);
    }
    std::cout << '\n';
    {
        vector<int> a{4,5,5,7,8};
        vector<int> b{2,4,5,6};
        print(a);
        print(b);

        vector<int> upper;
        set_difference(a.begin(), a.end(), b.begin(), b.end(), std::inserter(upper, upper.end()));
        print(upper);

        vector<int> lower;
        set_difference(b.begin(), b.end(), a.begin(), a.end(), std::inserter(lower, lower.end()));
        print(lower);
    }
    return 0;
}
```

## 结果

```shell
ASM generation compiler returned: 0
Execution build compiler returned: 0
Program returned: 0
8 7 5 4 
6 5 4 2 
4 5 7 8 
2 4 5 6 

4 5 7 8 
2 4 5 6 
7 8 
2 6 

4 5 5 7 8 
2 4 5 6 
5 7 8 
2 6 
```

## 结论

**Copies the elements from the sorted range [first1, last1) which are not found in the sorted range [first2, last2) to the range beginning at d_first. The output range is also sorted.**

接口限制输入容器需要时有序的，从测试结果看，只有set能保证得到正确结果
