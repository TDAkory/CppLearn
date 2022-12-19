# [makefile](https://makefiletutorial.com/#getting-started)

## 文件格式

### 概述

Makefile文件由一系列规则（rules）构成。每条规则的形式如下:

```makefile
<target> : <prerequisites> 
[tab]  <commands>
```

### 目标

* 一个目标（target）就构成一条规则。目标通常是文件名，指明Make命令所要构建的对象。

* 目标也可以是操作的名字，但当make发现，当前目录中，存在一个和该操作同名的文件时，make会认为构建目标已经存在了，不会重新构建，导致操作无法执行。这时应当声明操作名是一个“伪目标”：

```makefile
.PHONY: clean
clean:
    rm *.o temp
```

类似`PHONY`这样的内置目标名还有很多，参见[手册](https://www.gnu.org/software/make/manual/html_node/Special-Targets.html#Special-Targets)

### 前置条件

前置条件通常是一组文件名，之间用空格分隔。它指定了"目标"是否重新构建的判断标准：只要有一个前置文件不存在，或者有过更新（前置文件的last-modification时间戳比目标的时间戳新），"目标"就需要重新构建。

```makefile
result.txt: source.txt
    cp source.txt result.txt
```

### 命令

命令（commands）表示如何更新目标文件，由一行或多行的Shell命令组成。它是构建"目标"的具体指令，它的运行结果通常就是生成目标文件。

每行命令之前必须有一个tab键。如果想用其他键，可以用内置变量.RECIPEPREFIX声明。

```makefile
.RECIPEPREFIX = >
all:
> echo Hello, world
```

**每行命令在一个单独的shell中执行。这些Shell之间没有继承关系。**

```makefile
var-lost:
    export foo=bar
    echo "foo=[$$foo]"
```

上面代码执行后（make var-lost），取不到foo的值。因为两行命令在两个不同的进程执行。有三种解决办法，

```makefile
var-kept:
    export foo=bar; echo "foo=[$$foo]"

var-kept:
    export foo=bar; \
    echo "foo=[$$foo]"

.ONESHELL:
var-kept:
    export foo=bar; 
    echo "foo=[$$foo]"
```

## 语法

### 注释 

井号（#）在Makefile中表示注释。

### 回声

正常情况下，make会打印每条命令，然后再执行，这就叫做回声（echoing）。在命令的前面加上@，就可以关闭回声。

由于在构建过程中，需要了解当前在执行哪条命令，所以通常只在注释和纯显示的echo命令前面加上@。

### 通配符

Makefile 的通配符与 Bash 一致，主要有星号（*）、问号（？）和 [...] 。

### 模式匹配

Make命令允许对文件名，进行类似正则运算的匹配，主要用到的匹配符是%。使用匹配符%，可以将大量同类型的文件，只用一条规则就完成构建。

假定当前目录下有 f1.c 和 f2.c 两个源码文件:

```makefile
%.o: %.c

# 等同于下面的写法
f1.o: f1.c
f2.o: f2.c
```

### 变量和赋值符

允许使用等号自定义变量。

```makefile
txt = Hello World
test:
    @echo $(txt)
```

调用Shell变量，需要在美元符号前，再加一个美元符号，这是因为Make命令会对美元符号转义

```makefile
test:
    @echo $$HOME
```

变量的值可能指向另一个变量。

```makefile
VARIABLE = value
# 在执行时扩展，允许递归扩展。

VARIABLE := value
# 在定义时扩展。

VARIABLE ?= value
# 只有在该变量为空时才设置值。

VARIABLE += value
# 将值追加到变量的尾端。
```

### 内置变量

Make命令提供一系列内置变量，比如，$(CC) 指向当前使用的编译器，$(MAKE) 指向当前使用的Make工具。这主要是为了跨平台的兼容性，详细的内置变量清单见[手册](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html)。

### 自动变量

Make命令还提供一些自动变量，它们的值与当前规则有关。主要有以下几个。

* `$@` 指代当前目标
* `$<` 指代第一个前置条件
* `$?` 指代比目标更新的所有前置条件，之间以空格分隔
* `$^` 指代所有前置条件，之间以空格分隔
* `$*` 指代匹配符 % 匹配的部分， 比如% 匹配 f1.txt 中的f1 ，`$*` 就表示 f1。
* `$(@D)` 和 `$(@F)` 分别指向 `$@ `的目录名和文件名。比如，`$@`是 src/input.c，那么`$(@D)` 的值为 src ，`$(@F)` 的值为 input.c。
* `$(<D)` 和 `$(<F)` 分别指向 `$< `的目录名和文件名。

所有的自动变量清单，请看[手册](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html)

### 判断和循环

Makefile使用 Bash 语法，完成判断和循环。

```makefile
ifeq ($(CC),gcc)
  libs=$(libs_for_gcc)
else
  libs=$(normal_libs)
endif
```

### 函数

Makefile 还可以使用函数，格式如下。Makefile提供了许多[内置函数](https://www.gnu.org/software/make/manual/html_node/Functions.html)，可供调用。

```makefile
$(function arguments)
# 或者
${function arguments}
```
