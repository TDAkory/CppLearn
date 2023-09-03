# 日志操作

> doc/log_format.md

leveldb通过WAL来加速写入，同时提供写入保证。

* 可以将随机的写IO变成append，极大的提高写磁盘速度；
* 防止在节点down机导致内存数据丢失，造成数据丢失，这对系统来说是个灾难。

日志文件由32KB的块组成。唯一的例外是文件尾可能包含一个部分块。

```shell
    block := record* trailer?
    record :=
      checksum: uint32     // crc32c of type and data[] ; little-endian
      length: uint16       // little-endian
      type: uint8          // One of FULL, FIRST, MIDDLE, LAST
      data: uint8[length]
```

Log Type有4种：FULL = 1、FIRST = 2、MIDDLE = 3、LAST = 4。FULL类型表明该log record包含了完整的user record；而user record可能内容很多，超过了block的可用大小，就需要分成几条log record，第一条类型为FIRST，中间的为MIDDLE，最后一条为LAST。

由于一条logrecord长度最短为7，如果一个block的剩余空间<=6byte，那么将**被填充为*空字*符串，另外长度为7的log record是不包括任何用户数据的。