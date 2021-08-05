# learning-folly

## [facebook](https://github.com/facebook)/[**folly**](https://github.com/facebook/folly)

folly是Facebook开源库 的缩写，包含一系列核心库，很多时候都是作为其内部C++项目的依赖库，也是各个项目需要将代码共享时放置的地方，它是对boost和标准库的补充，而不是竞争关系。只有当其他库不存在需要的功能或性能无法满足需要时，才会开始进行开发新的组件。当标准库或boost废弃一些功能时，folly也会努力的跟着适应变化。

性能问题散布在folly库的各个角落，是其关注的核心点，有时甚至导致一些非常特别的设计\(例如，PackedSyncPtr.h，SmallLocks.h\)

folly的github地址是[https://github.com/facebook/folly](https://link.jianshu.com?t=https://github.com/facebook/folly),《modern c++ design》作者Andrei Alexandrescu是其主要作者之一。

### 逻辑设计

所有的符号都定义在顶层命名空间`folly`下，宏定义是例外，所有宏定义都是全大写字母，并以`FOLLY_`为前缀。

