# GTest框架核心测试宏详解：从基础到类型参数化

最近开发过程中需要用到支持参数、模板参数注入的测试套，对GTest进行了进一步研究学习，整理本文，旨在系统地解析GTest中最核心的几种测试宏，以根据不同的测试场景选择最合适的工具。

## 核心测试宏概览

| 宏 | 功能描述 | 适用场景 | 核心原理 |
| :--- | :--- | :--- | :--- |
| **TEST** | 定义最基础的独立测试用例。 | 简单的独立函数或功能验证，无需共享任何测试资源。 | 宏展开为独立的测试函数并注册到全局测试注册表。 |
| **TEST_F** | 在测试用例中共享固定测试夹具。 | 多个测试需要相同的初始化和清理逻辑（SetUp/TearDown）。 | 生成继承自指定夹具类的测试类，通过继承共享状态和方法。 |
| **TEST_P** | 参数化测试，用于同一逻辑的不同输入。 | 需要对多组数据进行重复测试（数据驱动测试）。 | 继承 `::testing::TestWithParam<T>`，运行时通过 `GetParam()` 获取参数。 |
| **TYPED_TEST** | 针对多种类型进行相同逻辑测试。 | 测试算法或数据结构在不同类型上的一致性（类模板场景）[reference:7]。 | 借助模板元编程和宏拼接，为类型列表中的每个类型生成独立的测试用例。 |
| **TYPED_TEST_P** | 可扩展的类型参数化测试。 | 需要在不同文件或模块中复用同一套类型化测试逻辑。 | 分两步：先注册一个“测试套件模板”，再通过 `INSTANTIATE_TYPED_TEST_SUITE_P` 用具体类型列表进行实例化。 |

## `TEST` 与 `TEST_F`

### `TEST`：独立的测试单元

`TEST` 是GTest中最基础的宏，用于定义一个无需共享环境的测试用例。测试体可以包含任何C++语句和GTest断言。

```cpp
#include <gtest/gtest.h>

int Factorial(int n) {
    // some implement
}

TEST(FactorialTest, HandlesZeroInput) {
  EXPECT_EQ(Factorial(0), 1);
}

TEST(FactorialTest, HandlesPositiveInput) {
  EXPECT_EQ(Factorial(1), 1);
  EXPECT_EQ(Factorial(3), 6);
}
```

逻辑上相关的 `TEST` 应该放在同一个测试套件（如 `FactorialTest`）中。

### `TEST_F`：测试套

当多个测试需要共享相同的设置和清理代码时，应使用 `TEST_F`。首先需要创建一个继承自 `::testing::Test` 的测试类。

```cpp
#include <gtest/gtest.h>
#include <queue>

template <typename T> class Queue {
    // some implement
};

class QueueTest : public ::testing::Test {
 protected:
  void SetUp() override {
    q1_.Enqueue(1);
    q2_.Enqueue(2); q2_.Enqueue(3);
  }
  void TearDown() override {}

  Queue<int> q0_; // 空队列
  Queue<int> q1_;
  Queue<int> q2_;
};

TEST_F(QueueTest, IsEmptyInitially) {
  EXPECT_EQ(q0_.size(), 0);
}

TEST_F(QueueTest, DequeueWorks) {
  int* n = q0_.Dequeue();
  EXPECT_EQ(n, nullptr);
  n = q1_.Dequeue();
  ASSERT_NE(n, nullptr); // 若失败，则终止当前函数
  EXPECT_EQ(*n, 1);
}
```

对于每个 `TEST_F`，框架都会：

1. 构造全新的测试对象
2. 调用 `SetUp()`
3. 运行测试体
4. 调用 `TearDown()`
5. 析构对象。因此测试之间是隔离的

## 参数化测试宏：`TEST_P`

`TEST_P` 用于编写数据驱动测试，即用同一段测试逻辑验证多组不同的输入数据。

1. **定义参数化测试套**：继承 `::testing::TestWithParam<T>`，其中 `T` 是参数类型。
2. **编写测试逻辑**：使用 `TEST_P` 宏，在测试体内通过 `GetParam()` 获取当前测试参数。
3. **实例化测试套件**：使用 `INSTANTIATE_TEST_SUITE_P` 宏提供具体的参数列表。

```cpp
#include <gtest/gtest.h>

bool IsPrime(int n) {
    // some implement
}

class PrimeTest : public ::testing::TestWithParam<int> {};

TEST_P(PrimeTest, ReturnsCorrectResult) {
  int n = GetParam();
  bool expected = (n == 2 || n == 3 || n == 5 || n == 7 || n == 11);
  EXPECT_EQ(IsPrime(n), expected);
}

INSTANTIATE_TEST_SUITE_P(
    PrimeValues,                     // 实例化名称（前缀）
    PrimeTest,                       // 测试夹具名
    ::testing::Values(1, 2, 3, 4, 5, 6, 7, 9, 11) // 参数生成器
);
```

GTest提供了多种参数生成器，如 `Values`、`Range`、`Bool`、`Combine` 等，用于灵活生成参数列表。

## 类型参数化测试宏：`TYPED_TEST` 与 `TYPED_TEST_P`

当需要测试模板代码或通用算法在多种类型上的行为时，就需要类型参数化测试。

### `TYPED_TEST`：简单的类型化测试

`TYPED_TEST` 适用于类型列表固定且测试逻辑定义在同一文件中的场景。

```cpp
#include <gtest/gtest.h>

template <typename T>
class ContainerTest : public ::testing::Test {
 protected:
  T container_;
};

using MyTypes = ::testing::Types<std::vector<int>, std::list<int>, std::deque<int>>;

TYPED_TEST_SUITE(ContainerTest, MyTypes); // 测试套 与 测试类型

TYPED_TEST(ContainerTest, IsEmptyInitially) {
  EXPECT_TRUE(this->container_.empty());
}
```

`TYPED_TEST` 会为 `MyTypes` 中的**每个类型**生成独立的测试用例。

### `TYPED_TEST_P`：可复用的类型化测试

`TYPED_TEST_P` 提供了更高的灵活性，允许将**测试逻辑的定义**与**类型的实例化**分离，便于在不同模块中复用同一套测试逻辑。

其使用流程分为五个步骤：

```cpp
// 1. 定义测试套模板
template <typename T>
class SmartPointerTest : public ::testing::Test {
 protected:
  std::unique_ptr<T> CreateValue(T val) { return std::make_unique<T>(val); }
};

// 2.声明套件，不绑定具体类型
TYPED_TEST_SUITE_P(SmartPointerTest);

// 3. 用 TYPED_TEST_P 定义测试用例
TYPED_TEST_P(SmartPointerTest, NotNullAfterCreation) {
  auto ptr = this->CreateValue(TypeParam{});
  EXPECT_NE(ptr, nullptr);
}
TYPED_TEST_P(SmartPointerTest, HoldsCorrectValue) {
  auto ptr = this->CreateValue(TypeParam{42});
  EXPECT_EQ(*ptr, TypeParam{42});
}

// 4. 注册所有测试用例到套件
REGISTER_TYPED_TEST_SUITE_P(SmartPointerTest,
    NotNullAfterCreation,
    HoldsCorrectValue
);

// 5. 实例化套件，绑定具体类型列表
using PointerTypes = ::testing::Types<int, double, std::string>;
INSTANTIATE_TYPED_TEST_SUITE_P(CommonTypes,  // 实例化前缀
                               SmartPointerTest, // 测试夹具模板
                               PointerTypes); // 要测试的具体类型列表
```

最终，GTest会为 `PointerTypes` 中的每个类型，生成所有已注册的测试。例如，会生成 `CommonTypes/SmartPointerTest/0.NotNullAfterCreation`（`int` 类型）等测试实例。

## 示例

```cpp
template <typename T>
class MyTestFixture : public ::testing::Test {
protected:
    T value_;
    
    void SetUp() override {
        value_ = T(42); // 初始化逻辑
    }
    
    void TearDown() override {
    }
};

TYPED_TEST_SUITE_P(MyTestFixture);

TYPED_TEST_P(MyTestFixture, TestSize) {
    EXPECT_GE(sizeof(this->value_), 1);
}

TYPED_TEST_P(MyTestFixture, TestDefaultValue) {
    EXPECT_EQ(this->value_, TypeParam(42));
}

TYPED_TEST_P(MyTestFixture, TestAssignment) {
    TypeParam new_value = TypeParam(100);
    this->value_ = new_value;
    EXPECT_EQ(this->value_, new_value);
}

REGISTER_TYPED_TEST_SUITE_P(MyTestFixture,
    TestSize,
    TestDefaultValue, 
    TestAssignment
);

using MyTypes = ::testing::Types<int, double, float, std::string>;

INSTANTIATE_TYPED_TEST_SUITE_P(MyPrefix, MyTestFixture, MyTypes);
```

对于上面的例子，gtest 会生成以下测试：

```shell
MyPrefix/MyTestFixture/0.TestSize        # int 类型
MyPrefix/MyTestFixture/0.TestDefaultValue
MyPrefix/MyTestFixture/0.TestAssignment

MyPrefix/MyTestFixture/1.TestSize        # double 类型  
MyPrefix/MyTestFixture/1.TestDefaultValue
MyPrefix/MyTestFixture/1.TestAssignment

MyPrefix/MyTestFixture/2.TestSize        # float 类型
MyPrefix/MyTestFixture/2.TestDefaultValue  
MyPrefix/MyTestFixture/2.TestAssignment

MyPrefix/MyTestFixture/3.TestSize        # std::string 类型
MyPrefix/MyTestFixture/3.TestDefaultValue
MyPrefix/MyTestFixture/3.TestAssignment
```

## 一些技巧

### **条件编译测试**

```cpp
#ifdef HAS_SPECIAL_FEATURE
using AllTypes = ::testing::Types<int, double, SpecialType>;
#else
using AllTypes = ::testing::Types<int, double>;
#endif

INSTANTIATE_TYPED_TEST_SUITE_P(AllTypesTest, MyTestFixture, AllTypes);
```

### **组合类型测试**

```cpp
template <typename T>
struct TypePair {
    using First = T;
    using Second = std::vector<T>;
};

using TestTypes = ::testing::Types<
    TypePair<int>,
    TypePair<double>,
    TypePair<std::string>
>;

INSTANTIATE_TYPED_TEST_SUITE_P(PairTests, MyTestFixture, TestTypes);
```

### 查看生成的测试名

```cpp
TYPED_TEST_P(MyTestFixture, DebugType) {
    std::cout << "Testing type: " << typeid(TypeParam).name() << std::endl;
}

// or

./your_test_binary --gtest_list_tests | grep "^[A-Za-z][a-zA-Z0-9_\/]*\.$"
```

### 选择性禁用测试

```cpp
TYPED_TEST_P(MyTestFixture, SomeTest) {
    if constexpr (std::is_same_v<TypeParam, ProblematicType>) {
        GTEST_SKIP() << "Skipped for ProblematicType";
    }
    // 正常测试逻辑
}
```

## 总结

| 宏 | 核心价值 | 选择依据 |
| :--- | :--- | :--- |
| **TEST** | 简单直接，无状态依赖。 | 测试独立函数，无需 Setup/Teardown。 |
| **TEST_F** | 共享夹具，避免重复初始化代码。 | 多个测试需要相同的初始状态或资源。 |
| **TEST_P** | **数据驱动**，一次编写，多数据验证。 | 同一逻辑需要应对多组不同的输入值。 |
| **TYPED_TEST** | **类型驱动**，为固定类型集生成测试。 | 测试模板代码在已知类型集上的行为。 |
| **TYPED_TEST_P** | **高度可复用**的类型测试框架。 | 测试逻辑需要被多个不同的类型列表复用。 |
