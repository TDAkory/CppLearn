# 优先考虑auto而不是显式类型声明

类型声明一个变量而不赋初值，编译是可以通过的，但是值是不确定的。它可能会被初始化为0，这得取决于工作环境。

### auto变量从初始化表达式中推导出类型，所以我们必须初始化。

```cpp
int x1;                         //潜在的未初始化的变量
	
auto x2;                        //错误！必须要初始化

auto x3 = 0;                    //没问题，x已经定义了

template<typename It>           //如之前一样
void dwim(It b,It e)
{
    while (b != e) {
        auto currValue = *b;
        …
    }
}
```

### 此外，auto还可以表示一些不方便书写，甚至是只有编译器才知道的类型。

```cpp
auto derefUPLess = 
    [](const std::unique_ptr<Widget> &p1,       //用于std::unique_ptr指向的Widget类型的
       const std::unique_ptr<Widget> &p2)       
    { return *p1 < *p2; };     

// c++14中，lambda表达式中的形参也可以使用auto
auto derefLess =                                //C++14版本
    [](const auto& p1,                          //被任何像指针一样的东西
       const auto& p2)                          //指向的值的比较函数
    { return *p1 < *p2; };
```

### auto具有直接持有闭包的能力

std::function是C++11标准库里的一个模板，它将函数指针通用化。函数指针只能指向函数，但是std::function对象可以指向任何callable对象(即可以像函数一样调用)。当你创建一个函数指针时，你必须给出函数的signature(即返回类型和形参表)，同样地，当你创建std::function对象时，你也必须指定。

因为lambda表达式产生callable对象，所以闭包(closure)可以被存在std::function对象中。这意味着我们可以声明C++11版本的不使用auto的derefUPLess：

```cpp
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)>
derefUPLess = [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2) {     
    return *p1 < *p2; 
};
```

一个持有闭包(closure)的auto变量具有和闭包(closure)一样的类型，并且因此仅消耗闭包(closure)所需求的内存空间。

一个持有闭包(closure)的std::function变量的类型是std::function模板的一个具现(instantiation)，并且它对于任意的函数signature都有固定的内存空间。这个内存空间的大小也许并不满足闭包(closure)的需求，所以std::function的构造函数可能会申请堆内存来存储闭包(closure)。因此，std::function对象通常会比auto对象消耗更多的内存空间。

另外，实现细节禁用inline，会导致间接地函数调用。因此，通过std::function对象调用闭包(closure)几乎肯定会比通过auto对象调用慢。

总之，std::function方法会比auto方法消耗更多空间且执行更慢，并且std::function方法还可能产生out-of-memory的异常。

### auto可以提供一种无须关系的类型捷径能力
```cpp
std::vector<int> v;
unsigned sz1 = v.size();
auto sz2 = v.size(); // sz2's type is std::vector<int>::size_type
```
v.size()的返回类型是std::vector<int>::size_type。std::vector<int>::size_type被指定为是unsigned int类型。所以很多人认为unsigned已经足够好了。在32位机子上，unsigned和std::vector<int>::size_type有相同的大小，但在64位机子上，unsigned是32位，而std::vector<int>::size_type是64位。所以可能在64位机子上会有错误的行为。使用auto就不会有这个问题。
```cpp
std::unordered_map<std::string, int> m;
...
 
for(const std::pair<std::string, int>& p : m)
{
    ... // do something with p
}
 
for(const auto& p : m)
{
    ... //as before
}
```
看出第一种写法有什么问题吗？std::unordered_map的key是const的，所以在hash table中的std::pair不是std::pair<std::string, int>，而是std::pair<const std::string, int>。对于第一种写法，编译器会想方设法把std::pair<const std::string, int>对象转换为std::pair<std::string, int>对象。编译器会拷贝m中的每个对象来生成一个临时对象，然后把p绑定到这个临时对象上。在每次循环结束时，临时对象会被销毁。这并不是你所期望的，你期望的是仅仅把p绑定到m中的每一个对象上。使用auto就可以避免这个问题。

这两个例子说明显式给出类型会导致你不期望的隐式转换。如果使用auto作为变量的类型，就不必担心你声明的类型与用来初始化变量的表达式的类型不一致的问题。
