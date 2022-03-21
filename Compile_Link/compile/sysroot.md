# sysroot

A sysroot is a directory which is considered to be the root directory for the purpose of **locating headers and libraries**. So for example if your build toolchain wants to find /usr/include/foo.h but you are cross-compiling and the appropriate foo.h is in /my/other/place/usr/include/foo.h, you would use /my/other/place as your sysroot.

- [gcc交叉编译时设置了“--sysroot“会产生哪些影响](https://blog.csdn.net/zvvzxzko2006/article/details/110467542)
- [交叉编译中的 --sysroot 等等在编译时的作用](https://www.cnblogs.com/welhzh/p/4702552.html)