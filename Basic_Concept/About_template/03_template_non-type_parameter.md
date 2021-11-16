# 非类型模板参数（template non-type parameter）

## 类模板参数

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

使用上述模板，需要同时指定元素的类型和个数。

```cpp
    Stack<int, 20> int20Stack;
```

同样的，非类型模板参数可以设置缺省值

```cpp
template <typename T = int, int MAXSIZE = 100>
class Stack {
    ...
};
```

## 函数模板参数

```cpp
template <typename T, int VAL>
T AddValue(T const& x)  //  用于把函数作为操作，示例如下
{
    return x + VAL;
};

std::transform(source.begin(), source.end(), dest.begin(), AddValue<int, 5>);    // 源集合中每个元素都加5，然后将值存入目标集合
```

但上述例子有个问题，AddValue<int, 5>是一个函数模板实例，而函数模板实例通常被看成是用来命名一组重载函数的集合（即使该组只有一个函数）。根据目前的标准，重载函数的集合不能被用于模板参数的演绎。因此需要将这个函数模板的实参强制类型转换为具体类型。

```cpp
std::transform(source.begin(), source.end(), dest.begin(), (int(*)(int const &))AddValue<int, 5>) 
```

## 非类型模板参数的限制

* 通常，可以是常整数、枚举值、指向外部链接对象的指针，浮点数和类对象是不允许作为非类型模板参数的。

* 字符串是内部链接对象，所以也不能作为模板实参

* 全局指针也不可以作为模板参数

```cpp
template <char const *name>
class MyClass {
    ...
};

MyClass<"hello"> x;     // ERROR!!!

extern char const s[] = "hello";

MyClass<s> x;   // OK，全局字符数组s由hello初始化，是一个外部链接对象
```
