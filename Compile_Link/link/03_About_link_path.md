# `rpath` & `runpath` & `LD_LIBRARY_PATH` & `LD_PRELOAD`

- [`rpath` & `runpath`](https://en.wikipedia.org/wiki/Rpath#:~:text=In%20computing%2C%20rpath%20designates%20the,(or%20another%20shared%20library).)
- [LD_LIBRARY_PATH](https://wiki.tcl-lang.org/page/LD_LIBRARY_PATH)
- [LD_PRELOAD](https://www.baeldung.com/linux/ld_preload-trick-what-is)
- [runpath和rpath的区别](https://segmentfault.com/a/1190000044513658)

## `rpath` 和 `runpath`

RPATH有两个比较相近的名称：rpath和runpath。这两个往往容易让人搞混。在最初的时候，ELF文件只有一个DT_RPATH标签用来表示rpath列表，后来ELF规范弃用了DT_RPATH，并引入了一个新的标签--DT_RUNPATH用于rpath列表。这两个标签的区别是它们相对于LD_LIBRARY_PATH环境变量的相对优先级。当链接器在程序运行时搜索动态库时，DT_RPATH有较高的优先级，DT_RUNPATH有较低的优先级。即搜索顺序如下：

1. DT_RPATH指定的目录列表。
2. 环境变量LD_LIBRARY_PATH指定的目录列表。
3. DT_RUNPATH指定的目录列表
4. /etc/ld.so.cache 缓存文件，通常包含 /etc/ld.so.conf 文件编译出的二进制列表（比如 CentOS 上，该文件会使用 include 从而使用 ld.so.conf.d 目录下面所有的 *.conf 文件，这些都会缓存在 ld.so.cache 中）
5. 系统的默认路径。如/lib,/usr/lib等。