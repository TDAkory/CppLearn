# 非类型模板参数（template non-type parameter）

接上一节的示例，可以用元素数目固定的数组类实现stack。但是开发者很难确认一个stack的最佳容量，因此可以让用户指定其大小

```cpp
template <typename T, int MAXSIZE>
class Stack {
    private:
        T elems[MAXSIZE];
        int numElems;

    public:
        Stack();
    ...
};

template <typename T, int MAXSIZE>
Stack<T, MAXSIZE>::Stack() : numElems(0) {}
```

MAXSIZE是第2个模板参数，类型为int，指定了数组最多可包含的元素个数。
