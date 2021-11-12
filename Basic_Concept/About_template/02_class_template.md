# Example

```cpp
template <typename T>
class Stack {
    private:
        std::vector<T> elems;
    
    public:
        Stack();
        void push(T const &);
        void pop();
        T top() cosnt;
}
```

这个类的类型是`Stack<T>`，其中T是模板参数，因此，当在声明中需要使用该类的类型时，必须使用`Stack<T>`。当使用类名而不是类的类型时，就应该只用`Stack`，比如构造和析构函数。

```cpp
template <typename T>
class Stack {
    ...
    Stack (Stack<T> const &);
    Stack<T> & operator= (Stack<T> const);
}
```

为了定义类模板的成员函数，必须指定该成员函数是一个函数模板，并且需要使用这个类模板的完整类型限定符。如下：

```cpp
template <typename T>
void Stack<T>::push (T const &elem) {
    elems.push_back(elem);
}
```

> 注意：vector的pop_back()只是删除末尾的元素，并没有返回该元素。之所以如此是充分考虑了异常安全性，因为要实现“一个绝对异常安全且返回被删除元素的pop()”是不可能的。（异常通常出现在copy construction期间，最简单的情况 内存不够，临时变量拷贝不出来，但是值丢失了。所以取元素和弹出元素设计成两步，保证即使有异常情况，数据不会丢失。）

因此用中间变量暂存一下，可以实现安全的`pop()`

```cpp
template <typename T>
T Stack<T>::pop() {
    if (elems.empty()) {
        throw std::out_of_range("empty stack");
    }
    T elem = elems.back();
    elems.pop_back();
    return elem;
}

template <typename T>
T Stack<T>::top() const {
    if (elems.empty()) {
        throw std::out_of_range("empty stack");
    }
    return elems.back();
} 
```

## 类模板的使用

* 使用类模板必须显式指定模板实参

* 只有被调用的成员函数，才会产生这些函数的实例化代码。即，对于类模板，成员函数只有在被使用的时候才会被实例化

* 如果类模板中含有静态成员，那么用来实例化的每种类型，都会实例化这些静态成员

## 类模板的特化

可以用模板实参来特化类模板，通过特化，可以优化基于某种类型的实现，或者客服某种特定类型在实例化类模板时的差异。

如果要特化一个类模板，需要特化改类模板的所有成员函数。

```cpp
template <>
class Stack<std::string> {
    ...
};
```

每个成员函数必须重新定义为普通函数，原来模板函数中的每个T也要相应地被特化类型取代：

```cpp
template <>
class Stack<std::string> {
    private:
        std::deque<std::string> elems;
    
    public:
        void push(std::string const &);
        void pop();
        std::string top() const;
        bool empty() const {
            return elems.empty();
        }
};

void Stack<std::string>::push(std::string const &elem) {
    elems.push_back(elem);
}

void Stack<std::string>::pop() {
    if (elems.empty()) {
        throw std::out_of_range("empty stack");
    }
    elems.pop_back();
} 

void Stack<std::string>::top() const {
    if (elems.empty()) {
        throw std::out_of_range("empty stack");
    }
    return elems.back();
}
```

* 特化的实现可以和基本类模板的实现完成不同

## 局部特化

```cpp
template <typename T1, typenmae T2>
class MyClass {
    ...
};

// 两个模板参数类型一致
template <typename T>
class MyClass<T, T> {};

// 第2个模板参数是int
template <typename T>
class MyClass<T, int> {};

// 两个模板参数是指针类型
template <typename T1, typenmae T2>
class MyClass<T1*, T2*> {};
```

如果有多个局部特化同等程度地匹配某个声明，那么久称该声明具有二义性

## 缺省模板实参

对于类模板，可以为模板参数定义缺省值：这些值被称为缺省模板参数，且可以引用之前的模板参数。

```cpp
// Example: 定义第2个模板参数，使用vector<>作为缺省值
#include <vector>
#include <stdexcept>

template <typename T, typename CONT = std::vectorM<T> >
class Stack {
    private:
        CONT elems;
    
    public:
        void push(T const&);
        void pop();
        T top() const;
        bool empty() const {
            return elems.empty();
        }
};

template <typename T, typename CONT>
void Stack<T, CONT>::push(T cosnt &elem) {
    elems.push_back(elem);
}

...


Stack<int> intStack; // 使用vector<int>
Stack<double, std::deque<double>> dblStack; // 使用deque<double>
```

## 小结

* 类模板：在类的实现中，可以有一个或多个类型没有被指定

* 使用类模板，需要传入某个具体类型作为模板参数，编译器会基于该类型来实例化模板

* 只有被调用的成员函数才会被实例化

* 可以用某种特定类型特化类模板

* 可以用某种特定类型局部特化类模板

* 可以为类模板参数定于缺省值，这些值可以引用之前的模板参数
