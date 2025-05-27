# When nanoseconds matter: ultrafast traging systems in C++

> [When Nanoseconds Matter: Ultrafast Trading Systems in C++ - David Gross - CppCon 2024](https://www.youtube.com/watch?v=sX2nF1fW7kI)

Summary of the talk:

Achieving low latency in a trading system cannot be an afterthought; it must be an integral part of the design from the very beginning. While low latency programming is sometimes seen under the umbrella of “code optimization”, the truth is that most of the work needed to achieve such latency is done upfront, at the design phase. How to translate our knowledge about the CPU and hardware into C++? How to use multiple CPU cores, handle concurrency issues and cost, and stay fast?

In this talk, I will be sharing with you some industry insights on how to design from scratch a low latency trading system. I will be presenting building blocks that application developers can directly re-use when in their trading systems (or some other high performance, highly concurrent applications).

Additionally, we will delve into several algorithms and data structures commonly used in trading systems, and discuss how to optimize them using the latest features available in C++. This session aims to equip you with practical knowledge and techniques to enhance the performance of your systems and make informed decisions about the tools and technologies you choose to employ.

> The order book is the heart of any trading system

## A way to running `perf` on a benchmark

```cpp
void RunPerf() {
    pid_t pid = fork();
    if (pid==0) {
        const auto parent_pid = std::to_string(getpid());
        std::cout << "Running perf on parent process " << parent_pid << std::endl;
        execlp("perf", "perf", ..., parent_pid.c_str(), (char *)nullptr);
        throw std::runtime_error("execlp failed");
    }
}

void InitAndRunBenchmark() {
    InitBenchmark(); // might take a long time
    RunPerf();
    RunBenchmark();
}
```

## Low-Lattency transport

* General pattern
  * kernel bypass when receiving data from the exchange(or other low-latency signals)
  * dispatch/fan-out to processes on the same server

## Principles

1. most of the time, you don't want node containers
2. understanding your problem
3. hand tailored(specilized) algorithms are key to achieve performance
4. simplicity is the ultimate sophistication
5. mechanical sympathy
6. true efficiency is dound not in the layers of complexity we add, but in the unnecessary layers we remove
7. choose the right tool for the right task
8. being fast is good, staying fast is better
9. thinking about the system as a whole
10. the performance of your code depends on your colleagues

