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