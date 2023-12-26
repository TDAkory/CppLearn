# [GoogleTest](https://github.com/google/googletest/blob/main/docs/index.md)

## Primer

## Advance

### `::testing::Environment`

```cpp
class Environment : public ::testing::Environment {
 public:
  ~Environment() override {}

  // Override this to define how to set up the environment.
  void SetUp() override {}

  // Override this to define how to tear down the environment.
  void TearDown() override {}
};
```

- global set-up and tear-down
- register an instance of your environment class with GoogleTest by calling the `::testing::AddGlobalTestEnvironment()` function
- multiple environment objects can be registered to one, and `SetUp` will be called in the registered order, `TearDown` in the reverse order