# [PImpl](https://en.cppreference.com/w/cpp/language/pimpl)

Removes implementation details of a class from its object representation by placing them in a separate class, accessed through an opaque pointer.

This technique is used to construct C++ library interfaces with stable ABI and to reduce compile-time dependencies.

```cpp
// --------------------
// interface (widget.h)
class widget
{
    // public members
private:
    struct impl; // forward declaration of the implementation class
    // One implementation example: see below for other design options and trade-offs
    std::experimental::propagate_const< // const-forwarding pointer wrapper
        std::unique_ptr<                // unique-ownership opaque pointer
            impl>> pImpl;               // to the forward-declared implementation class
};
 
// ---------------------------
// implementation (widget.cpp)
struct widget::impl
{
    // implementation details
};
```

Because private data members of a class participate in its object representation, affecting size and layout, and because private member functions of a class participate in overload resolution (which takes place before member access checking), any change to those implementation details requires recompilation of all users of the class.

pImpl breaks this compilation dependency; changes to the implementation do not cause recompilation. Consequently, if a library uses pImpl in its ABI, newer versions of the library may change the implementation while remaining ABI-compatible with older versions.

