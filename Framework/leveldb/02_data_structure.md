# 数据类型定义

![leveldb 基本框架](https://raw.githubusercontent.com/TDAkory/ImageResources/main/img/leveldb.png)

leveldb是一种基于operation log的文件系统，是Log-Structured-Merge Tree的典型实现。LSM源自Ousterhout和Rosenblum在1991年发表的经典论文<<The Design and Implementation of a Log-Structured File System >>。
## 一些约定

* leveldb对于数字采用little-endian存储
* 数字采用变长存储，VarInt，每个字节有效存储7bit，最高位的第8bit表示是否结束，如果是1则未结束
  * 在操作log中使用的定长存储
* 字符比较基于 unsigned char

## 基本数据结构

### Slice

类似string_view、go的slice

### Status

错误信息和返回状态的封装。为了节省空间Status并没有用std::string来存储错误信息，而是将返回码(code), 错误信息message及长度打包存储于一个字符串数组中。

成功状态OK 是NULL state_，否则state_ 是一个包含如下信息的数组:

```cpp
state_[0..3] == 消息message长度 

state_[4]    == 消息code

state_[5..]  == 消息message 
```

### Arena

内存池

### skip list

主要用于实现memtable

### Cache

双链表实现的LRUCache

### Others

Random、Hash、CRC32、Histogram
