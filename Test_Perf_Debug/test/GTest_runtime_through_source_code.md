# GTest Runtime through source code

> - [GoogleTest User’s Guide](https://google.github.io/googletest/)
> - [googletest github repo](https://github.com/google/googletest)
> 源码版本：v1.10.0

GoogleTest（通常缩写为GTest）作为由Google开发的C++测试框架，被广泛用于编写和运行C++测试，确保C++代码的质量和可靠性。这得益于GTest提供了一套完整的测试框架和工具，包括断言、测试套件、测试用例、测试运行器等。

本文假设读者已经了解并能使用Gtest。我们将从源码角度解析GTest的运行机制。

假设有一个简单的测试用例如下：

```cpp
#include <gtest/gtest.h>

TEST(MyTest, AddTest) {
  int result = 2 + 3;
  EXPECT_EQ(result, 5);
}
```

## `TEST`宏做了什么

```cpp
// gtest/include/gtest/gtest.h
#define TEST(test_suite_name, test_name) GTEST_TEST(test_suite_name, test_name)

#define GTEST_TEST(test_suite_name, test_name)             \
  GTEST_TEST_(test_suite_name, test_name, ::testing::Test, \
              ::testing::internal::GetTestTypeId())
```

GTEST_TEST_执行了如下操作：

```cpp
// gtest/include/gtest/internal/gtest-internal.h
// Helper macro for defining tests.
#define GTEST_TEST_(test_suite_name, test_name, parent_class, parent_id)      \
  static_assert(sizeof(GTEST_STRINGIFY_(test_suite_name)) > 1,                \
                "test_suite_name must not be empty");                         \
  static_assert(sizeof(GTEST_STRINGIFY_(test_name)) > 1,                      \
                "test_name must not be empty");                               \
  class GTEST_TEST_CLASS_NAME_(test_suite_name, test_name)                    \
      : public parent_class {                                                 \
   public:                                                                    \
    GTEST_TEST_CLASS_NAME_(test_suite_name, test_name)() {                    \
      ::testing::STECheckerHelper::CallAssertion = false;                     \
    }                                                                         \
    ~GTEST_TEST_CLASS_NAME_(test_suite_name, test_name)() {                   \
      ::testing::VerifyTest(__FILE__, __LINE__, #test_suite_name, #test_name, \
        ::testing::STECheckerHelper::CallAssertion);                          \
    }                                                                         \
                                                                              \
   private:                                                                   \
    virtual void TestBody();                                                  \
    static ::testing::TestInfo* const test_info_ GTEST_ATTRIBUTE_UNUSED_;     \
    GTEST_DISALLOW_COPY_AND_ASSIGN_(GTEST_TEST_CLASS_NAME_(test_suite_name,   \
                                                           test_name));       \
  };                                                                          \
                                                                              \
  ::testing::TestInfo* const GTEST_TEST_CLASS_NAME_(test_suite_name,          \
                                                    test_name)::test_info_ =  \
      ::testing::internal::MakeAndRegisterTestInfo(                           \
          #test_suite_name, #test_name, nullptr, nullptr,                     \
          ::testing::internal::CodeLocation(__FILE__, __LINE__), (parent_id), \
          ::testing::internal::SuiteApiResolver<                              \
              parent_class>::GetSetUpCaseOrSuite(__FILE__, __LINE__),         \
          ::testing::internal::SuiteApiResolver<                              \
              parent_class>::GetTearDownCaseOrSuite(__FILE__, __LINE__),      \
          new ::testing::internal::TestFactoryImpl<GTEST_TEST_CLASS_NAME_(    \
              test_suite_name, test_name)>);                                  \
  void GTEST_TEST_CLASS_NAME_(test_suite_name, test_name)::TestBody()
```

1. 宏展开校验了 test_suite_name 和 test_name 非空
2. 声明了一个测试类 `class test_suite_name##_##test_name##_Test ：public ::testing::Test`，
   1. 立即定义了其构造和析构函数
   2. 声明了一个虚函数 `TestBody()`
   3. 声明了一个静态成员变量 `::testing::TestInfo* const test_info_`
3. 在类外，利用 `MakeAndRegisterTestInfo` 初始化了静态成员变量 `test_info_`
4. 最后输出了 `TestBody()` 的函数签名，此时会连上我们测试用例的函数体，就像下面这样：

```cpp
void test_suite_name##_##test_name##_Test::TestBody() {
  int result = 2 + 3;
  EXPECT_EQ(result, 5);
}
```

目前为止，一个测试用例，就被转化为了一个测试类，其类名由测试套和测试用例的名字拼接而成，测试用例的函数体成为了其成员函数 `TestBody` 的函数体，非常的简洁清晰。

接下来我们看看 `MakeAndRegisterTestInfo` 具体做了什么：

```cpp
// src/gtest.cc
// Creates a new TestInfo object and registers it with Google Test;
// returns the created object.
//
// Arguments:
//
//   test_suite_name:   name of the test suite
//   name:             name of the test
//   type_param:       the name of the test's type parameter, or NULL if
//                     this is not a typed or a type-parameterized test.
//   value_param:      text representation of the test's value parameter,
//                     or NULL if this is not a value-parameterized test.
//   code_location:    code location where the test is defined
//   fixture_class_id: ID of the test fixture class
//   set_up_tc:        pointer to the function that sets up the test suite
//   tear_down_tc:     pointer to the function that tears down the test suite
//   factory:          pointer to the factory that creates a test object.
//                     The newly created TestInfo instance will assume
//                     ownership of the factory object.
TestInfo* MakeAndRegisterTestInfo(
    const char* test_suite_name, const char* name, const char* type_param,
    const char* value_param, CodeLocation code_location,
    TypeId fixture_class_id, SetUpTestSuiteFunc set_up_tc,
    TearDownTestSuiteFunc tear_down_tc, TestFactoryBase* factory) {
  TestInfo* const test_info =
      new TestInfo(test_suite_name, name, type_param, value_param,
                   code_location, fixture_class_id, factory);
  GetUnitTestImpl()->AddTestInfo(set_up_tc, tear_down_tc, test_info);
  return test_info;
}
```

可见其核心是构造了一个 TestInfo 结构，并将其连同 SetUpTestSuiteFunc TearDownTestSuiteFunc 一起注册。

这里的注册对象是 UnitTestImpl

```cpp
// googletest/src/gtest-internal-inl.h
// Convenience function for accessing the global UnitTest
// implementation object.
inline UnitTestImpl* GetUnitTestImpl() {
  return UnitTest::GetInstance()->impl();
}
```

被注册的对象是 TestInfo

```cpp
googletest/include/gtest/gtest.h
// A TestInfo object stores the following information about a test:
//
//   Test suite name
//   Test name
//   Whether the test should be run
//   A function pointer that creates the test object when invoked
//   Test result
//
// The constructor of TestInfo registers itself with the UnitTest
// singleton such that the RUN_ALL_TESTS() macro knows which tests to
// run.
class GTEST_API_ TestInfo {
  ...
};
```

顺着执行逻辑，注册动作的下钻如下：

```cpp
//googletest/src/gtest-internal-inl.h
  // Adds a TestInfo to the unit test.
  //
  // Arguments:
  //
  //   set_up_tc:    pointer to the function that sets up the test suite
  //   tear_down_tc: pointer to the function that tears down the test suite
  //   test_info:    the TestInfo object
  void AddTestInfo(internal::SetUpTestSuiteFunc set_up_tc,
                   internal::TearDownTestSuiteFunc tear_down_tc,
                   TestInfo* test_info) {
#if GTEST_HAS_FILE_SYSTEM
    ... // some support for thraed_safe death tests
#endif  // GTEST_HAS_FILE_SYSTEM

    GetTestSuite(test_info->test_suite_name_, test_info->type_param(),
                 set_up_tc, tear_down_tc)
        ->AddTestInfo(test_info);
  }
```

GetTestSuite 是一个具有 GetOrCreate 语义的接口，会得到一个 TestSuite 对象

```cpp
// googletest/src/gtest.cc
// Adds a test to this test suite.  Will delete the test upon
// destruction of the TestSuite object.
void TestSuite::AddTestInfo(TestInfo* test_info) {
  test_info_list_.push_back(test_info);
  test_indices_.push_back(static_cast<int>(test_indices_.size()));
}
```

到这里，一个测试用例的测试信息，就被保存了下来。

上述的宏展开，完成了测试类的定义，把开发写的用例函数体包装在测试类的 TestBody 成员函数中，最后为每个测试用例构造了一个测试信息记录 TestInfo ， 并缓存下来。看起来似乎完美了？其实还有一部分重要的内容，就是测试过程中的错误校验。

## 错误校验如何记录

我们在测试用例的编写中，常用 EXPECT_EQ 来描述对预期状态的校验。Gtest会在校验通过时继续执行，校验失败时输出信息并标记用例失败。那么 EXPECT_EQ 做了什么呢？

```cpp
// include/gtest/gtest.h
#define EXPECT_EQ(val1, val2) \
  EXPECT_PRED_FORMAT2(::testing::internal::EqHelper::Compare, val1, val2)
```

这里涉及两个内容，`::testing::internal::EqHelper::Compare`很显然是Gtest定义的比较函数, `EXPECT_PRED_FORMAT2`是对宏的进一步展开。

先看比较函数的定义：

```cpp
class EqHelper {
 public:
  // This templatized version is for the general case.
  template <
      typename T1, typename T2,
      // Disable this overload for cases where one argument is a pointer
      // and the other is the null pointer constant.
      typename std::enable_if<!std::is_integral<T1>::value ||
                              !std::is_pointer<T2>::value>::type* = nullptr>
  static AssertionResult Compare(const char* lhs_expression,
                                 const char* rhs_expression, const T1& lhs,
                                 const T2& rhs) {
    return CmpHelperEQ(lhs_expression, rhs_expression, lhs, rhs);
  }

  // With this overloaded version, we allow anonymous enums to be used
  // in {ASSERT|EXPECT}_EQ when compiled with gcc 4, as anonymous
  // enums can be implicitly cast to BiggestInt.
  //
  // Even though its body looks the same as the above version, we
  // cannot merge the two, as it will make anonymous enums unhappy.
  static AssertionResult Compare(const char* lhs_expression,
                                 const char* rhs_expression, BiggestInt lhs,
                                 BiggestInt rhs) {
    return CmpHelperEQ(lhs_expression, rhs_expression, lhs, rhs);
  }
  ...
};
```

两个重载实现的内部都调用了`CmpHelperEQ`，可以看到，它最终会调用入参类型的 `operator==` 来执行比较，相等或不等则分别由不同的辅助函数记录状态。

```cpp
// The helper function for {ASSERT|EXPECT}_EQ.
template <typename T1, typename T2>
AssertionResult CmpHelperEQ(const char* lhs_expression,
                            const char* rhs_expression, const T1& lhs,
                            const T2& rhs) {
  if (lhs == rhs) {
    return AssertionSuccess();
  }

  return CmpHelperEQFailure(lhs_expression, rhs_expression, lhs, rhs);
}
```

接下来看宏的进一步展开：

```cpp
// Binary predicate assertion macros.
#define EXPECT_PRED_FORMAT2(pred_format, v1, v2) \
  GTEST_PRED_FORMAT2_(pred_format, v1, v2, GTEST_NONFATAL_FAILURE_)

// Internal macro for implementing {EXPECT|ASSERT}_PRED_FORMAT2.
// Don't use this in your code.
#define GTEST_PRED_FORMAT2_(pred_format, v1, v2, on_failure)\
  GTEST_ASSERT_(pred_format(#v1, #v2, v1, v2), \
                on_failure)

// Internal macro for implementing {EXPECT|ASSERT}_PRED2.  Don't use
// this in your code.
#define GTEST_PRED2_(pred, v1, v2, on_failure)\
  GTEST_ASSERT_(::testing::AssertPred2Helper(#pred, \
                                             #v1, \
                                             #v2, \
                                             pred, \
                                             v1, \
                                             v2), on_failure)

// Helper function for implementing {EXPECT|ASSERT}_PRED2.  Don't use
// this in your code.
template <typename Pred,
          typename T1,
          typename T2>
AssertionResult AssertPred2Helper(const char* pred_text,
                                  const char* e1,
                                  const char* e2,
                                  Pred pred,
                                  const T1& v1,
                                  const T2& v2) {
  if (pred(v1, v2)) return AssertionSuccess();

  return AssertionFailure()
         << pred_text << "(" << e1 << ", " << e2
         << ") evaluates to false, where"
         << "\n"
         << e1 << " evaluates to " << ::testing::PrintToString(v1) << "\n"
         << e2 << " evaluates to " << ::testing::PrintToString(v2);
}

#define GTEST_ASSERT_(expression, on_failure) \
  GTEST_AMBIGUOUS_ELSE_BLOCKER_ \
  if (const ::testing::AssertionResult gtest_ar = \
      (SET_CALL_ASSERTION_TRUE, (expression))) \
    ; \
  else \
    on_failure(gtest_ar.failure_message())
```

这里的 `pred_format` 其实就是 `::testing::internal::EqHelper::Compare`。可以看到 `AssertPred2Helper` 把比较函数名、两个入参的名字、比较函数、两个入参都作为自身的入参，这也是我们在Gtest的错误日志中，可以同时看到参数名和参数值的原因。

从 `GTEST_ASSERT_` 的展开可以看到，当校验失败时，会调用 GTEST_NONFATAL_FAILURE_，因为我们展开的时 EXPECT_EQ, 所以错误是 non-fatal 的。那么 GTEST_NONFATAL_FAILURE_ 做了什么呢？

```cpp
#define GTEST_NONFATAL_FAILURE_(message) \
  GTEST_MESSAGE_(message, ::testing::TestPartResult::kNonFatalFailure)

#define GTEST_MESSAGE_(message, result_type) \
  GTEST_MESSAGE_AT_(__FILE__, __LINE__, message, result_type)

#define GTEST_MESSAGE_AT_(file, line, message, result_type) \
  ::testing::internal::AssertHelper(result_type, file, line, message) \
    = ::testing::Message()
```

这里错误信息，包含错误发生的文件名、行号，被包装成了一条信息，来构造一个`AssertHelper`对象：

```cpp
// googletest/src/gtest.cc
// Message assignment, for assertion streaming support.
void AssertHelper::operator=(const Message& message) const {
  UnitTest::GetInstance()->AddTestPartResult(
      data_->type, data_->file, data_->line,
      AppendUserMessage(data_->message, message),
      ShouldEmitStackTraceForResultType(data_->type)
          ? UnitTest::GetInstance()->impl()->CurrentOsStackTraceExceptTop(1)
          : ""
      // Skips the stack frame for this function itself.
  );  // NOLINT
}
```