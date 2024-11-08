# 编译和链接

## 预处理 Preprocessing

得到`.ii`的预处理文件，`gcc -E hello.c -o hello.i`

* 删除 #define，并展开所有的宏定义
* 处理条件编译指令
* 处理 #inlcude 预编译指令，将被包含的文件递归的插入到指令位置
* 删除所有注释
* 添加行号和文件名，便于产生调式信息
* 保留 #pragma 编译器指令

编译 Compilation

把预处理完的文件进行一系列的词法分析、语法分析、语义分析和优化后生成相应的汇编代码文件

`gcc -S hello.i -o hello.s`

汇编 Assembly

链接 Linking
