# GTest Runtime through source code

> - [GoogleTest User’s Guide](https://google.github.io/googletest/)
> - [googletest github repo](https://github.com/google/googletest)

`GoogleTest`（通常缩写为`GTest`）作为由Google开发的C++测试框架，被广泛用于编写和运行C++测试，确保C++代码的质量和可靠性。这得益于GTest提供了一套完整的测试框架和工具，包括断言、测试套件、测试用例、测试运行器等。

本文假设读者已经了解并能使用`GTest`。我们将从源码角度解析`GTest`的运行机制。

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

`GTEST_TEST_`执行了如下操作：

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

1. 宏展开校验了 `test_suite_name` 和 `test_name` 非空
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

可见其核心是构造了一个 `TestInfo` 结构，并将其连同 `SetUpTestSuiteFunc` `TearDownTestSuiteFunc` 一起注册。

这里的注册对象是 `UnitTestImpl`

```cpp
// googletest/src/gtest-internal-inl.h
// Convenience function for accessing the global UnitTest
// implementation object.
inline UnitTestImpl* GetUnitTestImpl() {
  return UnitTest::GetInstance()->impl();
}
```

被注册的对象是 `TestInfo`

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

`GetTestSuite` 是一个具有 `GetOrCreate` 语义的接口，会得到一个 `TestSuite` 对象

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

上述的宏展开，完成了测试类的定义，把开发写的用例函数体包装在测试类的 `TestBody` 成员函数中，最后为每个测试用例构造了一个测试信息记录 `TestInfo` ， 并缓存下来。看起来似乎完美了？其实还有一部分重要的内容，就是测试过程中的错误校验。

## 错误校验如何记录

我们在测试用例的编写中，常用 `EXPECT_EQ` 来描述对预期状态的校验。`GTest`会在校验通过时继续执行，校验失败时输出信息并标记用例失败。那么 `EXPECT_EQ` 做了什么呢？

```cpp
// include/gtest/gtest.h
#define EXPECT_EQ(val1, val2) \
  EXPECT_PRED_FORMAT2(::testing::internal::EqHelper::Compare, val1, val2)
```

这里涉及两个内容，`::testing::internal::EqHelper::Compare`很显然是`GTest`定义的比较函数, `EXPECT_PRED_FORMAT2`是对宏的进一步展开。

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

这里的 `pred_format` 其实就是 `::testing::internal::EqHelper::Compare`。可以看到 `AssertPred2Helper` 把比较函数名、两个入参的名字、比较函数、两个入参都作为自身的入参，这也是我们在`GTest`的错误日志中，可以同时看到参数名和参数值的原因。

从 `GTEST_ASSERT_` 的展开可以看到，当校验失败时，会调用 `GTEST_NONFATAL_FAILURE_`，因为我们展开的时 `EXPECT_EQ`, 所以错误是 `non-fatal` 的。那么 `GTEST_NONFATAL_FAILURE_` 做了什么呢？

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

`AddTestPartResult` 的核心动作可以分为三部分：构造错误消息（会根据入参来决定是否加入stack trace）、构造测试的部分结果并Report、根据测试的`TestPartResult::Type`进行一些错误处理

```cpp
// Adds a TestPartResult to the current TestResult object.  All Google Test
// assertion macros (e.g. ASSERT_TRUE, EXPECT_EQ, etc) eventually call
// this to report their results.  The user code should use the
// assertion macros instead of calling this directly.
void UnitTest::AddTestPartResult(TestPartResult::Type result_type,
                                 const char* file_name, int line_number,
                                 const std::string& message,
                                 const std::string& os_stack_trace)
    GTEST_LOCK_EXCLUDED_(mutex_) {
  Message msg;
  msg << message;

  ... // some operations to construct msg

  const TestPartResult result = TestPartResult(
      result_type, file_name, line_number, msg.GetString().c_str());
  impl_->GetTestPartResultReporterForCurrentThread()->ReportTestPartResult(
      result);

  ... // some error handling based on `result_type`
}

void DefaultPerThreadTestPartResultReporter::ReportTestPartResult(const TestPartResult& result) {
  unit_test_->GetGlobalTestPartResultReporter()->ReportTestPartResult(result);
}

void DefaultGlobalTestPartResultReporter::ReportTestPartResult(const TestPartResult& result) {
  unit_test_->current_test_result()->AddTestPartResult(result);
  unit_test_->listeners()->repeater()->OnTestPartResult(result);
}

void TestResult::AddTestPartResult(const TestPartResult& test_part_result) {
  test_part_results_.push_back(test_part_result);
}
```

完成了错误信息的记录，紧随而来的问题就是，这些信息是什么使用用到的，测试框架是如何读取这些信息，来给出最终的测试结果的。

```cpp
// Returns true if and only if the unit test passed (i.e. all test suites
// passed).
bool UnitTest::Passed() const { return impl()->Passed(); }

//#########################################

class UnitTestImpl {
  bool Passed() const { return !Failed(); }

  bool Failed() const {
    return failed_test_suite_count() > 0 || ad_hoc_test_result()->Failed();
  }
};

//#########################################

// Gets the number of successful test suites.
int UnitTestImpl::successful_test_suite_count() const {
  return CountIf(test_suites_, TestSuitePassed);
}

// Gets the number of failed test suites.
int UnitTestImpl::failed_test_suite_count() const {
  return CountIf(test_suites_, TestSuiteFailed);
}

//#########################################

// Returns true if and only if the test suite passed.
static bool TestSuitePassed(const TestSuite* test_suite) {
  return test_suite->should_run() && test_suite->Passed();
}

// Returns true if and only if the test suite failed.
static bool TestSuiteFailed(const TestSuite* test_suite) {
  return test_suite->should_run() && test_suite->Failed();
}

//#########################################

class TestSuite {

  // Returns true if and only if the test suite passed.
  bool Passed() const { return !Failed(); }

  // Returns true if and only if the test suite failed.
  bool Failed() const {
    return failed_test_count() > 0 || ad_hoc_test_result().Failed();
  }
};

// Gets the number of successful tests in this test suite.
int TestSuite::successful_test_count() const {
  return CountIf(test_info_list_, TestPassed);
}

// Gets the number of failed tests in this test suite.
int TestSuite::failed_test_count() const {
  return CountIf(test_info_list_, TestFailed);
}

//#########################################

// Returns true if and only if the test failed.
bool TestResult::Failed() const {
  for (int i = 0; i < total_part_count(); ++i) {
    if (GetTestPartResult(i).failed()) return true;
  }
  return false;
}

// Returns the i-th test part result among all the results. i can
// range from 0 to total_part_count() - 1. If i is not in that range,
// aborts the program.
const TestPartResult& TestResult::GetTestPartResult(int i) const {
  if (i < 0 || i >= total_part_count()) internal::posix::Abort();
  return test_part_results_.at(static_cast<size_t>(i));
}
```

从上面的调用链路可以看到，`UnitTest`会遍历检查其下属的 `TestSuite` 中的测试信息，来判断测试是否通过。

## 测试如何被执行

了解了测试用例的生成和错误信息的处理，最后我们再来看看测试是如何被执行的。从 `GTest` 的入口函数 `RUN_ALL_TEST()` 来看起：

```cpp
// googletest/include/gtest/gtest.h
inline int RUN_ALL_TESTS() { return ::testing::UnitTest::GetInstance()->Run(); }

// Runs all tests in this UnitTest object and prints the result.
// Returns 0 if successful, or 1 otherwise.
//
// We don't protect this under mutex_, as we only support calling it
// from the main thread.
int UnitTest::Run() {
  ... // GTEST_HAS_DEATH_TEST

  // Captures the value of GTEST_FLAG(catch_exceptions).  This value will be
  // used for the duration of the program.
  impl()->set_catch_exceptions(GTEST_FLAG_GET(catch_exceptions));

  ... // some set for different OS or Arch

  return internal::HandleExceptionsInMethodIfSupported(
             impl(), &internal::UnitTestImpl::RunAllTests,
             "auxiliary test code (environments or event listeners)")
             ? 0
             : 1;
}
```

从下面的代码段中可以看到：

* `UnitTestImpl::RunAllTests` 遍历其存储的测试用例信息，逐个执行 `GetMutableSuiteCase(test_index)->Run();`
* 在执行过程中，会判断用例是否需要跳过，是否存在全局的`Fatal`错误
* 在执行一个 `TestSuite` 内的用例时，还支持对用例顺序进行混淆，来排除用例之间的干扰
* 同时还能观察到针对 `TestSuite`，`Impl`在执行前后设置测试环境的动作

```cpp
// Runs all tests in this UnitTest object, prints the result, and
// returns true if all tests are successful.  If any exception is
// thrown during a test, the test is considered to be failed, but the
// rest of the tests will still be run.
//
// When parameterized tests are enabled, it expands and registers
// parameterized tests first in RegisterParameterizedTests().
// All other functions called from RunAllTests() may safely assume that
// parameterized tests are ready to be counted and run.
bool UnitTestImpl::RunAllTests() {
  ... // some preparations

  for (int i = 0; gtest_repeat_forever || i != repeat; i++) {
    // We want to preserve failures generated by ad-hoc test
    // assertions executed before RUN_ALL_TESTS().
    ClearNonAdHocTestResult();

    Timer timer;

    // Shuffles test suites and tests if requested.
    if (has_tests_to_run && GTEST_FLAG_GET(shuffle)) {
      random()->Reseed(static_cast<uint32_t>(random_seed_));
      // This should be done before calling OnTestIterationStart(),
      // such that a test event listener can see the actual test order
      // in the event.
      ShuffleTests();
    }

    // Tells the unit test event listeners that the tests are about to start.
    repeater->OnTestIterationStart(*parent_, i);

    // Runs each test suite if there is at least one test to run.
    if (has_tests_to_run) {
      // Sets up all environments beforehand. If test environments aren't
      // recreated for each iteration, only do so on the first iteration.
      if (i == 0 || recreate_environments_when_repeating) {
        repeater->OnEnvironmentsSetUpStart(*parent_);
        ForEach(environments_, SetUpEnvironment);
        repeater->OnEnvironmentsSetUpEnd(*parent_);
      }

      // Runs the tests only if there was no fatal failure or skip triggered
      // during global set-up.
      if (Test::IsSkipped()) {
        // Emit diagnostics when global set-up calls skip, as it will not be
        // emitted by default.
        TestResult& test_result =
            *internal::GetUnitTestImpl()->current_test_result();
        for (int j = 0; j < test_result.total_part_count(); ++j) {
          const TestPartResult& test_part_result =
              test_result.GetTestPartResult(j);
          if (test_part_result.type() == TestPartResult::kSkip) {
            const std::string& result = test_part_result.message();
            printf("%s\n", result.c_str());
          }
        }
        fflush(stdout);
      } else if (!Test::HasFatalFailure()) {
        for (int test_index = 0; test_index < total_test_suite_count();
             test_index++) {
          GetMutableSuiteCase(test_index)->Run();   // 执行测试用例
          if (GTEST_FLAG_GET(fail_fast) &&
              GetMutableSuiteCase(test_index)->Failed()) {
            for (int j = test_index + 1; j < total_test_suite_count(); j++) {
              GetMutableSuiteCase(j)->Skip();
            }
            break;
          }
        }
      } else if (Test::HasFatalFailure()) {
        // If there was a fatal failure during the global setup then we know we
        // aren't going to run any tests. Explicitly mark all of the tests as
        // skipped to make this obvious in the output.
        for (int test_index = 0; test_index < total_test_suite_count();
             test_index++) {
          GetMutableSuiteCase(test_index)->Skip();
        }
      }

      // Tears down all environments in reverse order afterwards. If test
      // environments aren't recreated for each iteration, only do so on the
      // last iteration.
      if (i == repeat - 1 || recreate_environments_when_repeating) {
        repeater->OnEnvironmentsTearDownStart(*parent_);
        std::for_each(environments_.rbegin(), environments_.rend(),
                      TearDownEnvironment);
        repeater->OnEnvironmentsTearDownEnd(*parent_);
      }
    }

    elapsed_time_ = timer.Elapsed();

    // Tells the unit test event listener that the tests have just finished.
    repeater->OnTestIterationEnd(*parent_, i);

    // Gets the result and clears it.
    if (!Passed()) {
      failed = true;
    }

    // Restores the original test order after the iteration.  This
    // allows the user to quickly repro a failure that happens in the
    // N-th iteration without repeating the first (N - 1) iterations.
    // This is not enclosed in "if (GTEST_FLAG(shuffle)) { ... }", in
    // case the user somehow changes the value of the flag somewhere
    // (it's always safe to unshuffle the tests).
    UnshuffleTests();

    if (GTEST_FLAG_GET(shuffle)) {
      // Picks a new random seed for each iteration.
      random_seed_ = GetNextRandomSeed(random_seed_);
    }
  }

  .. // some post-operations

  return !failed;
}
```

`TestSuite::Run` 则遍历执行每个用例，同时还提供了快速失败的逻辑分支，在该场景下，遇到失败会跳过后续用例的执行。

```cpp
// Runs every test in this TestSuite.
void TestSuite::Run() {
  ... 

  for (int i = 0; i < total_test_count(); i++) {
    if (skip_all) {
      GetMutableTestInfo(i)->Skip();
    } else {
      GetMutableTestInfo(i)->Run();
    }
    if (GTEST_FLAG_GET(fail_fast) &&
        GetMutableTestInfo(i)->result()->Failed()) {
      for (int j = i + 1; j < total_test_count(); j++) {
        GetMutableTestInfo(j)->Skip();
      }
      break;
    }
  }
  
  ...
}
```

这里就调用到了我们前面缓存下来的 `TestInfo`，它会通过工厂方法，来生成一个用户定义（宏展开的）`test_suite_test_name` 类，然后调用这个类的`Run`方法。

```cpp
// Creates the test object, runs it, records its result, and then
// deletes it.
void TestInfo::Run() {
  ...

  // Creates the test object.
  Test* const test = internal::HandleExceptionsInMethodIfSupported(
      factory_, &internal::TestFactoryBase::CreateTest,
      "the test fixture's constructor");

  // Runs the test if the constructor didn't generate a fatal failure or invoke
  // GTEST_SKIP().
  // Note that the object will not be null
  if (!Test::HasFatalFailure() && !Test::IsSkipped()) {
    // This doesn't throw as all user code that can throw are wrapped into
    // exception handling code.
    test->Run();
  }

  ...
}
```

在 `Run` 方法内可以看到，它通过一个工具函数`internal::HandleExceptionsInMethodIfSupported`，来按序调用这个类定义的 `Test::SetUp`, `Test::TestBody`, `Test::TearDown` ， 其中 `TestBody` 就是用户定义的测试用例的函数体了。

```cpp
// Runs the test and updates the test result.
void Test::Run() {
  if (!HasSameFixtureClass()) return;

  internal::UnitTestImpl* const impl = internal::GetUnitTestImpl();
  impl->os_stack_trace_getter()->UponLeavingGTest();
  internal::HandleExceptionsInMethodIfSupported(this, &Test::SetUp, "SetUp()");
  // We will run the test only if SetUp() was successful and didn't call
  // GTEST_SKIP().
  if (!HasFatalFailure() && !IsSkipped()) {
    impl->os_stack_trace_getter()->UponLeavingGTest();
    internal::HandleExceptionsInMethodIfSupported(this, &Test::TestBody,
                                                  "the test body");
  }

  // However, we want to clean up as much as possible.  Hence we will
  // always call TearDown(), even if SetUp() or the test body has
  // failed.
  impl->os_stack_trace_getter()->UponLeavingGTest();
  internal::HandleExceptionsInMethodIfSupported(this, &Test::TearDown,
                                                "TearDown()");
}
```

额外看一眼这个工具函数`internal::HandleExceptionsInMethodIfSupported`，可以看到它就是对是否捕获目标函数执行过程中抛出的异常的一个封装。

```cpp
// Runs the given method and catches and reports C++ and/or SEH-style
// exceptions, if they are supported; returns the 0-value for type
// Result in case of an SEH exception.
template <class T, typename Result>
Result HandleExceptionsInMethodIfSupported(T* object, Result (T::*method)(),
                                           const char* location) {
  // NOTE: The user code can affect the way in which Google Test handles
  // exceptions by setting GTEST_FLAG(catch_exceptions), but only before
  // RUN_ALL_TESTS() starts. It is technically possible to check the flag
  // after the exception is caught and either report or re-throw the
  // exception based on the flag's value:
  //
  // try {
  //   // Perform the test method.
  // } catch (...) {
  //   if (GTEST_FLAG_GET(catch_exceptions))
  //     // Report the exception as failure.
  //   else
  //     throw;  // Re-throws the original exception.
  // }
  //
  // However, the purpose of this flag is to allow the program to drop into
  // the debugger when the exception is thrown. On most platforms, once the
  // control enters the catch block, the exception origin information is
  // lost and the debugger will stop the program at the point of the
  // re-throw in this function -- instead of at the point of the original
  // throw statement in the code under test.  For this reason, we perform
  // the check early, sacrificing the ability to affect Google Test's
  // exception handling in the method where the exception is thrown.
  if (internal::GetUnitTestImpl()->catch_exceptions()) {
#if GTEST_HAS_EXCEPTIONS
    try {
      return HandleSehExceptionsInMethodIfSupported(object, method, location);
    } catch (const AssertionException&) {  // NOLINT
      // This failure was reported already.
    } catch (const internal::GoogleTestFailureException&) {  // NOLINT
      // This exception type can only be thrown by a failed Google
      // Test assertion with the intention of letting another testing
      // framework catch it.  Therefore we just re-throw it.
      throw;
    } catch (const std::exception& e) {  // NOLINT
      internal::ReportFailureInUnknownLocation(
          TestPartResult::kFatalFailure,
          FormatCxxExceptionMessage(e.what(), location));
    } catch (...) {  // NOLINT
      internal::ReportFailureInUnknownLocation(
          TestPartResult::kFatalFailure,
          FormatCxxExceptionMessage(nullptr, location));
    }
    return static_cast<Result>(0);
#else
    return HandleSehExceptionsInMethodIfSupported(object, method, location);
#endif  // GTEST_HAS_EXCEPTIONS
  } else {
    return (object->*method)();
  }
}
```

## 简单的总结

通过上面的分析，我们就基本搞明白了`GTest`的执行原理：

用户编写的测试用例，会通过`Test`宏展开成为一个`Test`类定义，同时会创建与其对应的 `TestSuite` `TestInfo` 信息，缓存到全局单例 `UnitTestImpl` 里面，测试用例的函数体会作为`Test::TestBody`的函数体。

在`main`函数中调用 `RUN_ALL_TEST()` 之后，会逐个执行每个用例的`TestBody`，把失败信息同步地记录到 `TestInfo` 包含着的 `TestResult` 中，在执行过程中 和 执行完所有用例后，都可以通过这些结果信息，来判断测试整体的成功状态。

至于对`ASSERT_XXX`的支持，则体现在 `Fatal-error` 的定义和判断逻辑上，对于用例环境的构建和清理，在上述执行过程中也有体现。