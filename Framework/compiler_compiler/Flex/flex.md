# [flex](https://en.wikipedia.org/wiki/Flex_(lexical_analyser_generator))

- [Flex, version 2.5](https://www.cs.princeton.edu/~appel/modern/c/software/flex/flex.html)

This manual describes flex, a tool for generating programs that perform pattern-matching on text. The manual includes both tutorial and reference sections

## Simple Example

1. 安装

```shell
sudo apt-get install flex
```

2. 编写.l文件

```c
# lexer.l
%{
#include <iostream>
%}

%%
[0-9]+    { std::cout << "Matched number: " << yytext << std::endl; }
[a-zA-Z]+ { std::cout << "Matched word: " << yytext << std::endl; }
%%

# main.cc
int main()
{
    yylex();
    return 0;
}
```

其中 `%{}` 用于定义一些 C++ 的头文件和变量，`%%` 之间是词法分析器的规则，以及相应的处理代码。上面的例子中，当匹配到一个数字或单词时，会输出匹配的内容，然后程序退出。

3. 生成对应代码 & 统一编译

```shell
flex -o lexer.cpp lexer.l

g++ -o program lexer.cpp main.cpp
```

## Some Tips

* By default, any text not matched by a flex scanner is copied to the output
* The flex input file consists of three sections, separated by a line with just `%%' in it
  
```flex
definitions
%%
rules
%%
user code
```

## Patterns

The patterns in the input are written using an extended set of regular expressions. These are:

