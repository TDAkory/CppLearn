# Gmock runtime through source code

> [googlemock](https://github.com/google/googletest/tree/main/googlemock)
> [gMock Cookbook](https://google.github.io/googletest/gmock_cook_book.html)

Google Mock（简称 gmock）是一个由 Google 开发的 C++ 单元测试框架，它扩展了 Google Test 测试框架，专门用于创建和使用模拟对象（mocks）。Google Mock 可以帮助你编写和运行单元测试，而不需要依赖外部的资源或服务，从而使得测试更加稳定和可靠。不过一个比较大的限制是gmock只能mock虚函数，具体的原因我们在之后的分析中逐步揭示。

还是从一个简单的例子开始分析：

```cpp
#include "gmock/gmock.h"
#include "gtest/gtest.h"

class Foo {
  public:
    virtual std::string getArbitraryString(int i) { ... }
};

class MockFoo : public Foo {
 public:
  MOCK_METHOD(std::string, getArbitraryString, (int), (override));
};

TEST(FooTest, TestGetArbitraryString) {
  MockFoo mock_foo;
  EXPECT_CALL(mock_foo, getArbitraryString())
      .WillOnce(testing::Return("Hello"))
      .WillRepeatedly(testing::Return("World"));
  
  EXPECT_EQ("Hello", mock_foo.getArbitraryString());
  EXPECT_EQ("World", mock_foo.getArbitraryString());
}
```

## `MOCK_METHOD`做了什么

```cpp
// googlemock/include/gmock/gmock-function-mocker.h
#define MOCK_METHOD(...)                                               \
  GMOCK_INTERNAL_WARNING_PUSH()                                        \
  GMOCK_INTERNAL_WARNING_CLANG(ignored, "-Wunused-member-function")    \
  GMOCK_PP_VARIADIC_CALL(GMOCK_INTERNAL_MOCK_METHOD_ARG_, __VA_ARGS__) \
  GMOCK_INTERNAL_WARNING_POP()
```

这里涉及 `INTERNAL_WARNING` 的宏是为了控制编译参数而设置的帮助宏，真正完成函数注册的是 `GMOCK_PP_VARIADIC_CALL(GMOCK_INTERNAL_MOCK_METHOD_ARG_, __VA_ARGS__)`

```cpp
// googlemock/include/gmock/internal/gmock-pp.h
// Calls CAT(_Macro, NARG(__VA_ARGS__))(__VA_ARGS__)
#define GMOCK_PP_VARIADIC_CALL(_Macro, ...) \
  GMOCK_PP_IDENTITY(                        \
      GMOCK_PP_CAT(_Macro, GMOCK_PP_NARG(__VA_ARGS__))(__VA_ARGS__))
```

由内向外分析这个宏，`GMOCK_PP_NARG` 是用来计算传入的参数个数的，在我们的例子中，`__VA_ARGS__`代表着`std::string, getArbitraryString, (int), (override)`:

```cpp
// Evaluates to the number of arguments after expansion.
//
//   #define PAIR x, y
//
//   GMOCK_PP_NARG() => 1
//   GMOCK_PP_NARG(x) => 1
//   GMOCK_PP_NARG(x, y) => 2
//   GMOCK_PP_NARG(PAIR) => 2
//
// Requires: the number of arguments after expansion is at most 15.
#define GMOCK_PP_NARG(...) \
  GMOCK_PP_INTERNAL_16TH(  \
      (__VA_ARGS__, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0))

#define GMOCK_PP_INTERNAL_16TH(_Args) \
  GMOCK_PP_IDENTITY(GMOCK_PP_INTERNAL_INTERNAL_16TH _Args)

// Because of MSVC treating a token with a comma in it as a single token when
// passed to another macro, we need to force it to evaluate it as multiple
// tokens. We do that by using a "IDENTITY(MACRO PARENTHESIZED_ARGS)" macro. We
// define one per possible macro that relies on this behavior. Note "_Args" must
// be parenthesized.
#define GMOCK_PP_INTERNAL_INTERNAL_16TH(_1, _2, _3, _4, _5, _6, _7, _8, _9, \
                                        _10, _11, _12, _13, _14, _15, _16,  \
                                        ...)                                \
  _16

// Returns the only argument.
#define GMOCK_PP_IDENTITY(_1) _1
```

这里巧妙的利用的宏的字符替换特性，在 `GMOCK_PP_INTERNAL_16TH` 的调用过程中，`__VA_ARGS__`所代表的参数占用了参数列表开头的几个位置，把 `15 ~ 0` 顺次‘挤’到了后面。接着在下一个调用`GMOCK_PP_INTERNAL_INTERNAL_16TH`的过程中，标记入参为 `1 ~ 16` ，这样前者被占用的参数位的个数，会体现在 `GMOCK_PP_INTERNAL_INTERNAL_16TH` 的第 16 个参数位置上，从下面的展开对比上，可以很清除的看到这个结果。

```shell
GMOCK_PP_INTERNAL_16TH(         std::string, getArbitraryString, (int), (override), 15, 14, 13, 12, 11,  10,  9,   8,   7,   6,    5,  4, 3, 2, 1, 0)

GMOCK_PP_INTERNAL_INTERNAL_16TH(  _1,           _2,                _3,      _4,     _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, ...)
```

> 从这两个宏展开，可以看到googlemock对入参个数的上限限制是16个，但实际上是远远用不到的，这一点在后面的分析可以看到

最终 `GMOCK_PP_NARG` 会返回参数个数 **4**， 所以我们的宏展开可以简化为：

```cpp
GMOCK_PP_IDENTITY(                        \
      GMOCK_PP_CAT(GMOCK_INTERNAL_MOCK_METHOD_ARG_, 4)(std::string, getArbitraryString, (int), (override)))
```

接着看一下 `GMOCK_PP_CAT`：

```cpp
// Expands and concatenates the arguments. Constructed macros reevaluate.
#define GMOCK_PP_CAT(_1, _2) GMOCK_PP_INTERNAL_CAT(_1, _2)

#define GMOCK_PP_INTERNAL_CAT(_1, _2) _1##_2
```

可以看到它完成的就是字符的拼接，所以上面的宏展开可以进一步简化成：

```cpp
GMOCK_PP_IDENTITY(GMOCK_INTERNAL_MOCK_METHOD_ARG_4(std::string, getArbitraryString, (int), (override)))
```

`GMOCK_INTERNAL_MOCK_METHOD_ARG_` 又是何许人也？从下面的宏定义可以看到：这个宏其实描述了mock函数的表达方式。

> `googlemock`支持的函数表达只有两种 `(_Ret, _MethodName, _Args)` 和 `(_Ret, _MethodName, _Args, _Spec)`，也就是最多支持4个入参，分别表示一个函数的：返回值、函数名、参数列表、const override等标记，远远用不到16个参数位置

```cpp
...

#define GMOCK_INTERNAL_MOCK_METHOD_ARG_2(...) \
  GMOCK_INTERNAL_WRONG_ARITY(__VA_ARGS__)

#define GMOCK_INTERNAL_MOCK_METHOD_ARG_3(_Ret, _MethodName, _Args) \
  GMOCK_INTERNAL_MOCK_METHOD_ARG_4(_Ret, _MethodName, _Args, ())

#define GMOCK_INTERNAL_MOCK_METHOD_ARG_4(_Ret, _MethodName, _Args, _Spec)  \
  GMOCK_INTERNAL_ASSERT_PARENTHESIS(_Args);                                \
  GMOCK_INTERNAL_ASSERT_PARENTHESIS(_Spec);                                \
  GMOCK_INTERNAL_ASSERT_VALID_SIGNATURE(                                   \
      GMOCK_PP_NARG0 _Args, GMOCK_INTERNAL_SIGNATURE(_Ret, _Args));        \
  GMOCK_INTERNAL_ASSERT_VALID_SPEC(_Spec)                                  \
  GMOCK_INTERNAL_MOCK_METHOD_IMPL(                                         \
      GMOCK_PP_NARG0 _Args, _MethodName, GMOCK_INTERNAL_HAS_CONST(_Spec),  \
      GMOCK_INTERNAL_HAS_OVERRIDE(_Spec), GMOCK_INTERNAL_HAS_FINAL(_Spec), \
      GMOCK_INTERNAL_GET_NOEXCEPT_SPEC(_Spec),                             \
      GMOCK_INTERNAL_GET_CALLTYPE_SPEC(_Spec),                             \
      GMOCK_INTERNAL_GET_REF_SPEC(_Spec),                                  \
      (GMOCK_INTERNAL_SIGNATURE(_Ret, _Args)))

#define GMOCK_INTERNAL_MOCK_METHOD_ARG_5(...) \
  GMOCK_INTERNAL_WRONG_ARITY(__VA_ARGS__)

...

#define GMOCK_INTERNAL_WRONG_ARITY(...)                                      \
  static_assert(                                                             \
      false,                                                                 \
      "MOCK_METHOD must be called with 3 or 4 arguments. _Ret, "             \
      "_MethodName, _Args and optionally _Spec. _Args and _Spec must be "    \
      "enclosed in parentheses. If _Ret is a type with unprotected commas, " \
      "it must also be enclosed in parentheses.")
```

我们继续分析 `GMOCK_INTERNAL_MOCK_METHOD_ARG_4` 的展开，它会执行以下的几个操作：

1. 校验参数列表是否在括号内
2. 校验函数的标识符是否在括号内
3. 校验函数签名是否符合规范
4. 校验函数的标识符是否符合规范
5. 最终通过 `GMOCK_INTERNAL_MOCK_METHOD_IMPL` 来完成mock函数的注册。

在解析`GMOCK_INTERNAL_MOCK_METHOD_IMPL`之前，我们先来看看`GMOCK_INTERNAL_MOCK_METHOD_IMPL`的几个入参分别是什么

1. `GMOCK_PP_NARG0 _Args` 的目的是解析mock函数的入参个数，注意这里并不是`MOCK_METHOD`宏的入参，而是我们被Mock的函数的真实入参，无参数会被标记为0，最多仍是支持15个参数，因为其内部使用的解析宏和前面解析宏参数的宏是一样的，都是`GMOCK_PP_NARG`，我们在上面已经分析过了。
2. `_MethodName` 就是被mock函数的函数名
3. 接下来是一组类似的宏，用来解析函数的标识符，我们分析其中一个`GMOCK_INTERNAL_HAS_OVERRIDE(_Spec)`，假设此时的 `_Spec = (const, override)`
   ```cpp
   #define GMOCK_INTERNAL_HAS_OVERRIDE(_Tuple) \
    GMOCK_PP_HAS_COMMA(                       \
      GMOCK_PP_FOR_EACH(GMOCK_INTERNAL_DETECT_OVERRIDE, ~, _Tuple))
   ```

   我们从外向内逐步展开这个宏，看看这里是如何进行的判断

   ```c
   GMOCK_PP_HAS_COMMA(GMOCK_PP_FOR_EACH(GMOCK_INTERNAL_DETECT_OVERRIDE, ~, (const, override)))
   ```

   ```c
   GMOCK_PP_INTERNAL_16TH((GMOCK_PP_FOR_EACH(GMOCK_INTERNAL_DETECT_OVERRIDE, ~, (const, override)), 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0))
   ```

   ```c
   GMOCK_PP_INTERNAL_16TH((GMOCK_PP_CAT(GMOCK_PP_INTERNAL_FOR_EACH_IMPL_, GMOCK_PP_NARG0 (const, override))(0, GMOCK_INTERNAL_DETECT_OVERRIDE, ~, (const, override)), 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0))
   ```

   这里`GMOCK_PP_NARG0`计算了标识符的个数，得到2，然后`GMOCK_PP_CAT`完成了字符串的拼接，因此得到了

   ```c
   GMOCK_PP_INTERNAL_16TH(GMOCK_PP_INTERNAL_FOR_EACH_IMPL_2(0, GMOCK_INTERNAL_DETECT_OVERRIDE, ~, (const, override)),1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0)
   ```

   中间的FOR_EACH部分，会把标识符展开为两个表达式

   ```c
   GMOCK_INTERNAL_DETECT_OVERRIDE(0, ~, const)
   GMOCK_INTERNAL_DETECT_OVERRIDE(1, ~, override)
   ```

   两者分别对应着`GMOCK_INTERNAL_DETECT_OVERRIDE_const`, `,`

   ```c
   // Returns 1 if the expansion of arguments has an unprotected comma. Otherwise
   // returns 0. Requires no more than 15 unprotected commas.
   GMOCK_PP_INTERNAL_16TH(GMOCK_INTERNAL_DETECT_OVERRIDE_const, , ,1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0)
   ```

   **简单来说，这一组宏展开通过将要检测的目标对象转换为参数列表中的`,`，又由于宏展开时，逗号会作为宏参数的分隔符。因此多一个逗号意味着参数个数增加了两个。最终通过检测变换后的参数列表中，是否存在‘未受到保护的逗号’，来判断是否命中了检测目标。**

然后我们看下`GMOCK_INTERNAL_MOCK_METHOD_IMPL`的展开，看起来显得比较复杂：

```cpp
#define GMOCK_INTERNAL_MOCK_METHOD_IMPL(_N, _MethodName, _Constness,           \
                                        _Override, _Final, _NoexceptSpec,      \
                                        _CallType, _RefSpec, _Signature)       \
  typename ::testing::internal::Function<GMOCK_PP_REMOVE_PARENS(               \
      _Signature)>::Result                                                     \
  GMOCK_INTERNAL_EXPAND(_CallType)                                             \
      _MethodName(GMOCK_PP_REPEAT(GMOCK_INTERNAL_PARAMETER, _Signature, _N))   \
          GMOCK_PP_IF(_Constness, const, )                                     \
              _RefSpec _NoexceptSpec GMOCK_PP_IF(_Override, override, )        \
                  GMOCK_PP_IF(_Final, final, ) {                               \
    GMOCK_MOCKER_(_N, _Constness, _MethodName)                                 \
        .SetOwnerAndName(this, #_MethodName);                                  \
    return GMOCK_MOCKER_(_N, _Constness, _MethodName)                          \
        .Invoke(GMOCK_PP_REPEAT(GMOCK_INTERNAL_FORWARD_ARG, _Signature, _N));  \
  }                                                                            \
  ::testing::MockSpec<GMOCK_PP_REMOVE_PARENS(_Signature)> gmock_##_MethodName( \
      GMOCK_PP_REPEAT(GMOCK_INTERNAL_MATCHER_PARAMETER, _Signature, _N))       \
      GMOCK_PP_IF(_Constness, const, ) _RefSpec {                              \
    GMOCK_MOCKER_(_N, _Constness, _MethodName).RegisterOwner(this);            \
    return GMOCK_MOCKER_(_N, _Constness, _MethodName)                          \
        .With(GMOCK_PP_REPEAT(GMOCK_INTERNAL_MATCHER_ARGUMENT, , _N));         \
  }                                                                            \
  ::testing::MockSpec<GMOCK_PP_REMOVE_PARENS(_Signature)> gmock_##_MethodName( \
      const ::testing::internal::WithoutMatchers&,                             \
      GMOCK_PP_IF(_Constness, const, )::testing::internal::Function<           \
          GMOCK_PP_REMOVE_PARENS(_Signature)>*) const _RefSpec _NoexceptSpec { \
    return ::testing::internal::ThisRefAdjuster<GMOCK_PP_IF(                   \
        _Constness, const, ) int _RefSpec>::Adjust(*this)                      \
        .gmock_##_MethodName(GMOCK_PP_REPEAT(                                  \
            GMOCK_INTERNAL_A_MATCHER_ARGUMENT, _Signature, _N));               \
  }                                                                            \
  mutable ::testing::FunctionMocker<GMOCK_PP_REMOVE_PARENS(_Signature)>        \
  GMOCK_MOCKER_(_N, _Constness, _MethodName)

  #define GMOCK_MOCKER_(arity, constness, Method) \
  GTEST_CONCAT_TOKEN_(gmock##constness##arity##_##Method##_, __LINE__)
```

第一部分：生成了一个和原函数的函数名相同的函数，用来替换生产代码的真实逻辑。函数内会调用mocker的`SetOwnerAndName`把Mock类指针和函数名注册进去，然后Invoke方法是为了后续调用用户指定的EXPECT_CALL中的逻辑的
第二部分：生成了两个 gmock_原函数名 的新函数。这个函数返回了一个参数筛选的函数对象，会被EXPECT_CALL调用，用来记录用户设置的期望行为
第三部分：生成了一个唯一的变量名：gmock + 是否const + 参数个数 + 方法名 + 行号

这里用到了一个类型 `::testing::FunctionMocker`，这是一个模板类。其中R是被Mock函数的返回值，Args是参数列表。在模板类的开头就定义了被Mock的函数原型。

在Invoke方法中，会寻找特定参数下，能够匹配到的Action，然后执行。

```cpp
// googlemock/include/gmock/gmock-spec-builders.h
template <typename R, typename... Args>
class FunctionMocker<R(Args...)> final : public UntypedFunctionMockerBase {
  using F = R(Args...);

 public:
  using Result = R;
  using ArgumentTuple = std::tuple<Args...>;
  using ArgumentMatcherTuple = std::tuple<Matcher<Args>...>;

  ...

  // Returns the result of invoking this mock function with the given
  // arguments.  This function can be safely called from multiple
  // threads concurrently.
  Result Invoke(Args... args) GTEST_LOCK_EXCLUDED_(g_gmock_mutex) {
    return InvokeWith(ArgumentTuple(std::forward<Args>(args)...));
  }

  MockSpec<F> With(Matcher<Args>... m) {
    return MockSpec<F>(this, ::std::make_tuple(std::move(m)...));
  }

 ...
};  // class FunctionMocker
```

那么显而易见的问题就是，这些候选的Action是如何被注册进来的。这就涉及到`EXPECT_CALL`的逻辑了。

## `EXPECT_CALL`做了什么