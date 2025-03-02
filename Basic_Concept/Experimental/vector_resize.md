# resize时发生了什么

在 C++ 中，std::vector 的 resize 方法的行为取决于你是增加还是减少容器的大小，以及容器中元素的类型。具体来说：

1. 增加容器大小

当使用 resize 增加 std::vector 的大小时，新元素的初始化方式取决于以下几点：

* 如果元素类型有默认构造函数：新元素将通过调用默认构造函数进行初始化。
* 如果元素类型没有默认构造函数：新元素将被默认初始化（对于基本数据类型，这意味着值初始化为零；对于类类型，这意味着调用默认构造函数）。
  
2. 减少容器大小

当使用 resize 减少 std::vector 的大小时，超出新大小的元素将被销毁，但不会调用默认构造函数。减少大小的操作不会创建新对象，因此不会涉及构造函数的调用。

```cpp
// https://godbolt.org/z/hWzG91xWr

#include <iostream>
#include <vector>

class MyClass {
public:
    MyClass() {
        std::cout << "Default constructor called\n";
    }

    MyClass(int value) : value_(value) {
        std::cout << "Constructor with value called\n";
    }

    MyClass(const MyClass& o) {
        std::cout << "copy Constructor with value called\n";
        value_ = o.value_;
    }

    ~MyClass() {
        std::cout << "Destructor called\n";
    }

private:
    int value_;
};

int main() {
    std::vector<MyClass> vec;

    std::cout << "Resizing to 3 elements (increase)\n";
    vec.resize(3);

    std::cout << "Resizing to 1 element (decrease)\n";
    vec.resize(1); 

    std::cout << "Resizing to 5 elements with default value\n";
    vec.resize(5, MyClass(42));  

    std::cout << "end" << std::endl;
    return 0;
}
```

```shell
Resizing to 3 elements (increase)
Default constructor called
Default constructor called
Default constructor called
Resizing to 1 element (decrease)
Destructor called
Destructor called
Resizing to 5 elements with default value
Constructor with value called
copy Constructor with value called
copy Constructor with value called
copy Constructor with value called
copy Constructor with value called
copy Constructor with value called
Destructor called
Destructor called
end
Destructor called
Destructor called
Destructor called
Destructor called
Destructor called
```