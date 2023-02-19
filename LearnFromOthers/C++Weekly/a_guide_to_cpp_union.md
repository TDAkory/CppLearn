# [Union](https://www.youtube.com/watch?v=Lu1WsdQOi0E)

* unions can be anonymous
* only one member is active at a time
* you can only access the currently active member
  * any other access IS undefined behavior
* No member is the default member
* Unions can have constructors and destructors in C++11!
  * but destructors are almost impossible to get correct!
* Unions support regular member functions
* we need manual bookkeeping to know the active member
* there is a paper for compile-time querying active member
  * https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2641r1.html
* Use std::optional or std::variant if that's what you want!

```cpp
int main() {
    union {
        int i;
        float f;
    };

    i = 42;
}
```