# Item 10: Prefer scoped enums to unscoped enums

C++98风格的`enum`都是`unscoped enum`
```cpp
enum Color { black, white, red };   //black, white, red在Color所在的作用域
auto white = false;                 //错误! white早已在这个作用域中声明
```
限域`enum`是通过`enum class`声明，所以它们有时候也被称为枚举类(enum classes)
```cpp
enum class Color { black, white, red }; //black, white, red限制在Color域内
auto white = false;                     //没问题，域内没有其他“white”

Color c = white;                        //错误，域中没有枚举名叫white

Color c = Color::white;                 //没问题
auto c = Color::white;                  //也没问题（也符合Item5的建议）
```
* 限域`enum`可以减少命名空间污染
* 限域`enum`在作用域内是强类型，未限域`enum`中的枚举名会隐式的转换为整形，
```cpp
enum Color { black, white, red };       //未限域enum

std::vector<std::size_t>                //func返回x的质因子
  primeFactors(std::size_t x);

Color c = red;
…

if (c < 14.5) {                         // Color与double比较 (!)
    auto factors =                      // 计算一个Color的质因子(!)
      primeFactors(c);
    …
}

```