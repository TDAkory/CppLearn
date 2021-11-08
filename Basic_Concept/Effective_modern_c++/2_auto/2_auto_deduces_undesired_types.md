Item 6: Use the explicitly typed initializer idiom when auto deduces undesired types

# auto推导不符合预期，使用显式类型初始化的惯用法

## `vector<bool>`的特例

```cpp
std::vector<bool> features(const Widget& w);

Widget w;
…
bool highPriority = features(w)[5];     //w高优先级吗？
processWidget(w, highPriority);         //根据它的优先级处理w


auto highPriority = features(w)[5];     //w高优先级吗？
processWidget(w,highPriority);          //未定义行为！,highPriority包含一个悬置指针！

```
虽然从概念上来说std::vector<bool>意味着存放bool，但是std::vector<bool>的operator[]不会返回容器中元素的引用（这就是std::vector::operator[]可返回除了bool以外的任何类型），取而代之它返回一个std::vector<bool>::reference的对象（一个嵌套于std::vector<bool>中的类）,[【详见】](https://en.cppreference.com/w/cpp/container/vector_bool)

具体的
```cpp
bool highPriority = features(w)[5];     //显式的声明highPriority的类型
```
这里，features返回一个std::vector<bool>对象后再调用operator[]，operator[]将会返回一个std::vector<bool>::reference对象，然后再通过隐式转换赋值给bool变量highPriority。highPriority因此表示的是features返回的std::vector<bool>中的第五个bit，这也正如我们所期待的那样。

```cpp
auto highPriority = features(w)[5];     //推导highPriority的类型
```
同样的，features返回一个std::vector<bool>对象，再调用operator[]，operator[]将会返回一个std::vector<bool>::reference对象，但是现在这里有一点变化了，auto推导highPriority的类型为std::vector<bool>::reference，但是highPriority对象没有第五bit的值。

std::vector<bool>::reference是一个代理类（proxy class）的例子：所谓代理类就是以模仿和增强一些类型的行为为目的而存在的类。很多情况下都会使用代理类，std::vector<bool>::reference展示了对std::vector<bool>使用operator[]来实现引用bit这样的行为。

**不可见的代理类通常不适用于auto**

显式类型初始器惯用法使用auto声明一个变量，然后对表达式强制类型转换（cast）得出你期望的推导结果。举个例子，我们该怎么将这个惯用法施加到highPriority上？
```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

## 请记住：

* 不可见的代理类可能会使auto从表达式中推导出“错误的”类型
* 显式类型初始器惯用法强制auto推导出你想要的结果