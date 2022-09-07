# [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#main)

## P.1: Express ideas directly in code

```cpp
class Date {
public:
    Month month() const;  // do
    int month();          // don't
    // ...
};
```