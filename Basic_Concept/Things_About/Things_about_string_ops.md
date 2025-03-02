# Tips about string operations

## String concatenation

有些场景下字符串的拼接可能不够高效

```cpp
std::string foo = LongString1();
std::string bar = LongString2();
std::string baz = LongString3();

// OK
std::string foobar = foo + bar;
// Maybe expensive
std::string foobarbaz = foo + bar + baz;
```

由于 C++ 中没有三参数运算符的重载，这个操作必然需要调用两次 `string::operator+`。

```cpp
std::string temp = foo + bar;
std::string foobarbaz = std::move(temp) + baz;
```

在这两次调用之间，操作会构造（并存储）一个临时字符串，这可能涉及两次内存拷贝。C++11 至少允许第二次连接操作无需创建一个新的字符串对象：std::move(temp) + baz 等同于 std::move(temp.append(baz))。然而，有可能最初为临时字符串分配的缓冲区不足以容纳最终的字符串，这种情况下就需要重新分配内存（并再次复制）。因此，在最坏的情况下，n 个字符串连接操作可能需要 O(n) 次重新分配。

在多个字符串拼接的场景下，可以考虑使用：

* `absl::StrCat()` `absl::StrAppend()`
* `boost::algorithm::join`
* `std::ostringstream`
* `folly::fbstring`

## String split

use `absl::StrSplit()`, powerful

```cpp
// Splits on commas. Stores in vector of string_view (no copies).
std::vector<absl::string_view> v = absl::StrSplit("a,b,c", ',');

// Splits on commas. Stores in vector of string (data copied once).
std::vector<std::string> v = absl::StrSplit("a,b,c", ',');

// Splits on literal string "=>" (not either of "=" or ">")
std::vector<absl::string_view> v = absl::StrSplit("a=>b=>c", "=>");

// Splits on any of the given characters (',' or ';')
using absl::ByAnyChar;
std::vector<std::string> v = absl::StrSplit("a,b;c", ByAnyChar(",;"));

// Stores in various containers (also works w/ absl::string_view)
std::set<std::string> s = absl::StrSplit("a,b,c", ',');
std::multiset<std::string> s = absl::StrSplit("a,b,c", ',');
std::list<std::string> li = absl::StrSplit("a,b,c", ',');

// Equiv. to the mythical SplitStringViewToDequeOfStringAllowEmpty()
std::deque<std::string> d = absl::StrSplit("a,b,c", ',');

// Yields "a"->"1", "b"->"2", "c"->"3"
std::map<std::string, std::string> m = absl::StrSplit("a,1,b,2,c,3", ',');
```

## String format

* absl::Substitute
* std::format

## String join

`absl::StrJoin()`

## Ref

* [Tip of the Week #3: String Concatenation and operator+ vs. StrCat()](https://abseil.io/tips/3)