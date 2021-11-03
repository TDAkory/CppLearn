## Example
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

