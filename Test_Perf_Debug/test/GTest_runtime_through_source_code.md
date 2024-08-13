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