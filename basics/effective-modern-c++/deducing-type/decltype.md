# decltype

decltype typically parrots back the exact type of the name or expression you give it:

```cpp
const int i = 0;             // decltype(i) is const int
bool f(const Widget &w);     // decltype(w) is const Widget &, decltype(f) is bool(const Widget &)

struct Point {
    int x, y;    // decltype(Point::x) is int 
};               // decltype(Point::y) is int

Widget w;        // decltype(w) is Widget

if (f(w)) ...    // decltype(f(w)) is bool

template<typename T>
class vector {
public:
    ...
    T& operator[](std::size_t index);
    ...
};

vector<int> v;     // decltype(v) is vector<int>

if (v[0] == 0) ... // decltype(v[0]) is int&
```

In C++11, perhaps the primary use for decltype is declaring function templates where the function’s return type depends on its parameter types.

```cpp
// c++11 works, need refinement
// The use of auto before the function name has nothing to do with type deduction. 
// Rather, it indicates that C++11’s trailing return type syntax is being used
template<typename Container, typename Index>
auto authAndAccess(Container &c, Index i) 
    -> decltype(c[i])
{
    authenticateUser();
    return c[i];
}

// c++14 works, need refinement
// In particular, for the common case where c[i] returns a T&, 
// authAndAccess will also return a T&
template<typename Container, typename Index>
decltype(auto)
authAndAccess(Container& c, Index i)
{
  authenticateUser();
  return c[i];
}
```

上述实现无法接收一个右值参数

```cpp
template<typename Container, typename Index>    //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}

template<typename Container, typename Index>    //最终的C++11版本
auto
authAndAccess(Container&& c, Index i)
->decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

## Things to Remember

* decltype almost always yields the type of a variable or expression without any modifications.
* For lvalue expressions of type T other than names, decltype always reports a type of T&.
* C++14 supports decltype\(auto\), which, like auto, deduces a type from its initializer, but it performs the type deduction using the decltype rules.

