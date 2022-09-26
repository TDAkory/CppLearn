# Compile Time Trace

## Time report

可以使用`-ftime-report`编译选项来生成单个文件的编译耗时报告单。

```shell
clang++ -c main.cpp -ftime-report

g++ -c main.cpp -ftime-report
```

clang相比于gcc有更加丰富的输出， 优化的级别越高， 构建的时间就越长， 输出的信息就越多。

## Time trace

`time-report`统计的时间主要是针对编译过程中的阶段性时间的统计， 无法和源码结合起来， 不容易定位项目中存在编译耗时的代码位置。 `time-trace`是将编译时间的信息输出到json文件中， 这有利于扩展到平台上或工具中进行更加友好的数据展示， 同时它不仅可以追踪编译过程的阶段时间， 而且能够定位到哪个文件， 哪个函数。

生成json数据

```shell
clang++ -c main.cpp -ftime-trace
```

在chrome中加载

```shell
open chrome://tracing/
load xxx.json
```

[speedscope](https://github.com/jlfwong/speedscope)加载

```shell
open https://www.speedscope.app
Browse
```
