# Future Direction

## 编译器对连续尖括号的支持

低版本编译器会混淆模板的尖括号和右移运算符，需要在使用时添加空格，高版本编译器则没有这项困扰

```cpp
typedef std::vector<std::list<int> > LineTable;     // 低版本编译器正确

typedef std::vector<std::list<int>> OtherTable;     // 低版本错误、高版本正确
```

