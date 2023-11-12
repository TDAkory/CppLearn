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

## 写日志

> db/log_writer.h

使用了内存映射技术。

* type_crc_数组存放了预计算的，所有可支持类型的CRC32值

```cpp
Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  size_t left = slice.size();

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  Status s;
  bool begin = true;
  // 循环写，直到成功写入全部数据，或者出错
  do {
    const int leftover = kBlockSize - block_offset_;
    assert(leftover >= 0);
    // 查看当前block是否剩余空间 < 7，小于则补零，换一个新的block写入，并重置偏移
    if (leftover < kHeaderSize) {
      // Switch to a new block
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        static_assert(kHeaderSize == 7, "");
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      block_offset_ = 0;
    }

    // Invariant: we never leave < kHeaderSize bytes in a block.
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;
    const size_t fragment_length = (left < avail) ? left : avail;

    RecordType type;
    const bool end = (left == fragment_length);
    if (begin && end) {
      type = kFullType;
    } else if (begin) {
      type = kFirstType;
    } else if (end) {
      type = kLastType;
    } else {
      type = kMiddleType;
    }

    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    left -= fragment_length;
    begin = false;
  } while (s.ok() && left > 0);
  return s;
}
```