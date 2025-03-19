# Random Trap

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

在cppreference中仅找到这样一句描述：

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

* 创建 std::default_random_engine 对象 engine，使用默认构造函数，它会使用实现定义的默认种子。
* 如果每次使用默认构造函数创建 std::default_random_engine 对象，可能会得到相同的随机数序列（取决于实现）。为了每次运行程序得到不同的随机数序列，可以使用 std::random_device 作为种子

