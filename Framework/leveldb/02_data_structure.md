# 数据类型定义

## 一些约定

* leveldb对于数字采用little-endian存储
* 数字采用变长存储，VarInt，每个字节有效存储7bit，最高位的第8bit表示是否结束，如果是1则未结束
  * 在操作log中使用的定长存储
* 字符比较基于 unsigned char

## Memtable

