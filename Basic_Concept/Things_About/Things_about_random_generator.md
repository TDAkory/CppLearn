# Random Trap

## 从问题出发

事情的来源是如下一小段代码，其基本设想是在服务发现的节点中随机选择几个作为本地的缓存

```cpp
// get endpoints by service discovery
auto endpoints = XXXDiscovery();

// shffule 
std::shuffle(endpoints.begin(), endpoints.end(), std::default_random_engine());

// get batch_size as local cache
for (int i = 0; i < batch_size; ++i) {
    result[i].ip = endpoints[i].ip;
    result[i].port = endpoints[i].port;
}

return result;
```

在测试阶段发现若干进程通过这段逻辑发现的下游节点都是一样的，明显不符合预期。

在`cppreference`中找到这样一句描述：

`default_random_engine(C++11)  an implementation-defined RandomNumberEngine type`

通过[一段代码](https://godbolt.org/z/bv3fffK4r)进行验证

```cpp
#include <iostream>
#include <random>

int main() {
    {   
        std::cout << "default construct: ";
        std::default_random_engine engine;
        for (int i = 0; i < 5; ++i) {
            std::cout << engine() << " ";
        }
        std::cout << std::endl;
    }

    {   
        std::cout << "default construct: ";
        std::default_random_engine engine;
        for (int i = 0; i < 5; ++i) {
            std::cout << engine() << " ";
        }
        std::cout << std::endl;
    }

    {   
        std::cout << "construct with seed 42: ";
        std::default_random_engine engine(42);
        for (int i = 0; i < 5; ++i) {
            std::cout << engine() << " ";
        }
        std::cout << std::endl;
    }

    {   
        std::cout << "construct with seed 132: ";
        std::default_random_engine engine(132);
        for (int i = 0; i < 5; ++i) {
            std::cout << engine() << " ";
        }
        std::cout << std::endl;
    }

    return 0;
}
```

得到的结果如下

```shell
default construct: 16807 282475249 1622650073 984943658 1144108930 
default construct: 16807 282475249 1622650073 984943658 1144108930 
construct with seed 42: 705894 1126542223 1579310009 565444343 807934826 
construct with seed 132: 2218524 779510869 1588928583 1163544036 698523470 
```

* 创建 `std::default_random_engine` 对象 engine，使用默认构造函数，它会使用实现定义的默认种子。
* 如果每次使用默认构造函数创建 `std::default_random_engine` 对象，可能会得到相同的随机数序列（取决于实现）。为了每次运行程序得到不同的随机数序列，可以使用 `std::random_device` 作为种子

## 更多关于随机数生成的陷阱

### 未初始化随机数种子

随机数生成函数 [`rand()`](https://en.cppreference.com/w/c/numeric/random/rand) 依赖于一个种子值来初始化随机数序列。如果种子值固定或者没有正确初始化，每次程序运行时生成的随机数序列都会相同，这就失去了随机性。例如，若使用 `srand(0)` 或者不调用 `srand()` 函数（此时随机数种子会被设置为1），程序每次运行都会生成相同的随机数序列。

```cpp
#include <iostream>
#include <cstdlib>

int main() {
    for (int i = 0; i < 5; ++i) {
        std::cout << rand() << std::endl;
    }
    return 0;
}
```

通常的做法是使用当前时间作为种子值，通过 time(0) 函数获取当前时间：

```cpp
#include <iostream>
#include <cstdlib>
#include <ctime>

int main() {
    srand(static_cast<unsigned int>(time(0)));

    for (int i = 0; i < 5; ++i) {
        std::cout << rand() << std::endl;
    }
    return 0;
}
```

`rand()` 函数基于线性同余算法，其随机范围通常较小（`RAND_MAX` 一般为 32767），周期较短，生成的随机数序列规律性较强。这使得生成的随机数在某些对随机性要求较高的场景下不适用，例如密码学领域。

#### 分布函数使用不当

在使用分布函数时，如果对其边界条件和特性理解不准确，可能会导致生成的随机数不符合预期。例如，`std::uniform_int_distribution` 是闭区间，若需要生成 `[a, b)` 范围的随机数，没有进行正确的调整，就会出现问题。

```cpp
// https://godbolt.org/z/xE3WPY3zd
#include <iostream>
#include <random>

int main() {
    std::random_device rd;
    std::mt19937 gen(rd());
    // 错误使用，未考虑闭区间问题
    std::uniform_int_distribution<> dis(1, 10);
    for (int i = 0; i < 5; ++i) {
        std::cout << dis(gen) << std::endl;
    }
    return 0;
}
```

一种可能得结果：

```shell
3
3
2
10
4
```

正确的做法是：

1. 将上界设为略小于 b 的值，如 `double upper = std::nextafter(10, std::numeric_limits<double>::min()); // 取 b 的前一个可表示数`
2. 或者在生成值后判断是否小于 b，否则重新生成

### 伪随机数在安全场景下不够安全

伪随机数生成器（PRNG）使用确定的数学算法产生具备良好统计属性的数字序列，但实际上这种数字序列并非具备真正的随机特性。伪随机数生成器通常以一个种子值为起始，每次计算使用当前种子值生成一个输出及一个新种子，这个新种子会被用于下次计算。在安全场景中，如果种子值可以被预测，那么生成的随机数序列也可以被预测，从而导致安全风险。

因此可以使用 `std::random_device` 获取安全的随机种子，然后结合 `std::mt19937` 或其他随机数引擎生成随机数。

也可以使用一些更专业、高性能或适用于特定场景的随机数生成库，可满足高精度、高安全性或特殊分布需求：

* [CUDA Curand](https://docs.nvidia.com/cuda/curand/): 用于 GPU 加速的随机数生成库，提供高性能的随机数生成。
* [PCG（Permuted Congruential Generator）](http://www.pcg-random.org/): 比标准库的 mt19937 更快，且具有更好的统计特性（如抗碰撞性）
* [Botan](https://botan.randombit.net/): 一个功能强大的密码学库，包含多种随机数生成器和安全工具
* [GSL](https://www.gnu.org/software/gsl/): GNU 科学库，提供了各种随机数生成器和统计工具

### 多线程环境下的竞争问题

在多线程环境中，如果多个线程共享同一个随机数引擎（如 `std::mt19937` `std::linear_congruential_engine` 等），会导致线程安全问题，即数据竞争。不同线程可能会同时修改随机数引擎的状态，从而破坏随机数的生成逻辑。

```cpp
#include <iostream>
#include <random>
#include <thread>
#include <vector>

std::mt19937 gen;   // 不是线程安全的

void threadFunction() {
    std::uniform_int_distribution<> dis(1, 10);
    for (int i = 0; i < 5; ++i) {
        std::cout << std::this_thread::get_id() << ": " << dis(gen) << std::endl;
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 3; ++i) {
        threads.emplace_back(threadFunction);
    }
    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

可以考虑用 `thread_local` 存储随机数引擎，或使用互斥锁进行保护

## 一些最佳实践

### 使用 `<random>` 库

更灵活、更强大，更多引擎和分布函数的支持。

```cpp
#include <iostream>
#include <random>

int main() {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(1, 10);
    for (int i = 0; i < 5; ++i) {
        std::cout << dis(gen) << std::endl;
    }
    return 0;
}
```

### 正确初始化种子

使用 `std::random_device` 作为种子初始化随机数引擎，它通常基于硬件随机源，能提供更真实的随机性。同时，种子只需要初始化一次，避免在循环中重复初始化。


### 正确使用分布函数

在使用分布函数时，要明确其边界条件和特性。如果需要特定范围的随机数，要进行正确的调整。

```cpp
#include <iostream>
#include <random>

int main() {
    std::random_device rd;
    std::mt19937 gen(rd());

    double upper = std::nextafter(10, std::numeric_limits<double>::min()); // 取 b 的前一个可表示数
    std::uniform_int_distribution<> dis(1, upper);
    for (int i = 0; i < 5; ++i) {
        std::cout << dis(gen) << std::endl;
    }
    return 0;
}
```

### 考虑性能优化

避免在循环中频繁初始化种子，预先生成一批随机数并缓存起来，以提高性能。

### 加密场景使用安全随机数

在加密场景中，使用加密安全的随机数生成器，如 `std::random_device`（若系统支持硬件随机源）或操作系统提供的接口（如 Linux 的 `/dev/urandom`）。

```
// https://godbolt.org/z/dK6xYbYbo
#include <iostream>
#include <random>
#include <fstream>

// 从 /dev/urandom 读取随机数
void generateSecureKey() {
    std::ifstream urandom("/dev/urandom", std::ios::binary);
    if (urandom) {
        unsigned char key[16];
        urandom.read(reinterpret_cast<char*>(key), sizeof(key));
        urandom.close();
        std::cout << "Generated secure key: ";
        for (int i = 0; i < sizeof(key); ++i) {
            std::cout << std::hex << static_cast<int>(key[i]);
        }
        std::cout << std::endl;
    }
}

int main() {
    generateSecureKey();
    return 0;
}

// Generated secure key: f558692b6149b4bb445e8d45896e967
```
### 多线程环境下的处理

在多线程环境中，为每个线程创建独立的随机数引擎实例，避免线程安全问题。

### 验证随机数的分布均匀性

对于对随机性要求较高的场景，使用统计测试（如卡方检验）来验证随机数的分布均匀性和独立性。

```cpp
// https://godbolt.org/z/5oGb6Tq4q
#include <iostream>
#include <random>
#include <vector>

int main() {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(1, 10);
    std::vector<int> frequency(10, 0);
    for (int i = 0; i < 1000; ++i) {
        int num = dis(gen);
        frequency[num - 1]++;
    }
    for (int i = 0; i < 10; ++i) {
        std::cout << "Number " << i + 1 << " frequency: " << frequency[i] << std::endl;
    }
    return 0;
}
```
