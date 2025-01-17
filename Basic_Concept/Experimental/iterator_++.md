# Iterator and ++

## demo

```cpp
#include <iostream>
#include <algorithm>
 
template<long FROM, long TO>
class Range {
public:
    // member typedefs provided through inheriting from std::iterator
    class iterator: public std::iterator<
                        std::input_iterator_tag,   // iterator_category
                        long,                      // value_type
                        long,                      // difference_type
                        const long*,               // pointer
                        long                       // reference
                                      >{
        long num = FROM;
    public:
        explicit iterator(long _num = 0) : num(_num) {}
        iterator& operator++() {num = TO >= FROM ? num + 1: num - 1; return *this;}
        iterator operator++(int) {iterator retval = *this; ++(*this); return retval;}
        bool operator==(iterator other) const {return num == other.num;}
        bool operator!=(iterator other) const {return !(*this == other);}
        reference operator*() const {return num;}
    };
    iterator begin() {return iterator(FROM);}
    iterator end() {return iterator(TO >= FROM? TO+1 : TO-1);}
};
 
int main() {
    Range<1,10> r;
    std::cout << *(r.begin()++) << " " << *(++r.begin()) << std::endl;
    {
        Range<1,10>::iterator ex_it;
        for(auto it = r.begin(); it!=r.end(); ex_it = it++) {
            std::cout << *it << " " << *ex_it << std::endl;
        }
    }
    {
        Range<1,10>::iterator ex_it;
        for(auto it = r.begin(); it!=r.end(); ex_it = ++it) {
        std::cout << *it << " " << *ex_it << std::endl;
        }
    }
    {
        for(auto it = r.begin(); it!=r.end(); ++it) {
            std::cout << *it << " ";
        }
        std::cout << std::endl;
    }
    {
        for(auto it = r.begin(); it!=r.end(); it++) {
            std::cout << *it << " ";
        }
        std::cout << std::endl;
    }
    {
        for(auto it = r.begin(); it!=r.end(); it = ++it) {
            std::cout << *it << " ";
        }
        std::cout << std::endl;
    }
    {
        Range<1,10>::iterator ex_it;
        for(auto it = r.begin(); it!=r.end(); it = it++) {
            std::cout << *it << " "; 
        }
        std::cout << std::endl;
    }
    
    std::cout << '\n';
}
```

## 结果

```shell
ASM generation compiler returned: 0
Execution build compiler returned: 0
Program returned: 143
1 2
1 0
2 1
3 2
4 3
5 4
6 5
7 6
8 7
9 8
10 9
1 0
2 2
3 3
4 4
5 5
6 6
7 7
8 8
9 9
10 10
1 2 3 4 5 6 7 8 9 10 
1 2 3 4 5 6 7 8 9 10 
1 2 3 4 5 6 7 8 9 10 
1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1
[Truncated]
```

## 结论

* for循环中，除非存在显式的赋值运算符，否则 ++iter 和 iter++，编译器执行的都是先增

