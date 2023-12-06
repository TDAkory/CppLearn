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

`EmitPhysicalRecord` 实际写入

首先计算header，并Append到log文件，一共7byte大小，格式为

```cpp
| CRC32 (4 byte) | payload length lower + high (2 byte) | type (1byte)|

```

然后写入payload，并flush，更新block的当前偏移。

```cpp
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr,
                                  size_t length) {
  assert(length <= 0xffff);  // Must fit in two bytes
  assert(block_offset_ + kHeaderSize + length <= kBlockSize);

  // Format the header
  char buf[kHeaderSize];
  buf[4] = static_cast<char>(length & 0xff);
  buf[5] = static_cast<char>(length >> 8);
  buf[6] = static_cast<char>(t);

  // Compute the crc of the record type and the payload.
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, length);
  crc = crc32c::Mask(crc);  // Adjust for storage
  EncodeFixed32(buf, crc);

  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, length));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  block_offset_ += kHeaderSize + length;
  return s;
}
```

## 读日志

> db/log_reader.h

Reader主要使用了两个类：

* Reporter：负责错误上报
  * Corruption 接口
* SequentialFile：负责文件读取
  * Read接口
  * Skip接口

Reader包含一些成员变量表示读取状态

```cpp
  Slice buffer_;
  bool eof_;  // Last Read() indicated EOF by returning < kBlockSize

  // Offset of the last record returned by ReadRecord.
  uint64_t last_record_offset_;
  // Offset of the first location past the end of buffer_.
  uint64_t end_of_buffer_offset_;

  // Offset at which to start looking for the first record to return
  uint64_t const initial_offset_;
```

读的主要逻辑在`bool ReadRecord(Slice* record, std::string* scratch)`中

```cpp
// Read the next record into *record.  Returns true if read
  // successfully, false if we hit end of the input.  May use
  // "*scratch" as temporary storage.  The contents filled in *record
  // will only be valid until the next mutating operation on this
  // reader or the next mutation to *scratch.
  bool ReadRecord(Slice* record, std::string* scratch);
```

### 1

首先根据 `initial_offset_` 来定位读取位置，如果 `当前偏移 < 指定偏移`，则需要通过 `SkipToInitialBlock` 进行跳转:

```cpp
bool Reader::SkipToInitialBlock() {
  // 计算在block内的偏移，以及block的起始位置
  const size_t offset_in_block = initial_offset_ % kBlockSize;
  uint64_t block_start_location = initial_offset_ - offset_in_block;

  // Don't search a block if we'd be in the trailer
  if (offset_in_block > kBlockSize - 6) {
    block_start_location += kBlockSize;
  }
  // 准备读取的block的起始位置
  end_of_buffer_offset_ = block_start_location;

  // Skip to start of first block that can contain the initial record
  if (block_start_location > 0) {
    Status skip_status = file_->Skip(block_start_location);
    if (!skip_status.ok()) {
      ReportDrop(block_start_location, skip_status);
      return false;
    }
  }

  return true;
}
```

### 2

然后再循环读取之前，初始化一些局部变量

```cpp
  // 当前是否在fragment内，也就是遇到了FIRST 类型的record
  bool in_fragmented_record = false;
  // Record offset of the logical record that we're reading
  // 0 is a dummy value to make compilers happy
  uint64_t prospective_record_offset = 0;
```

### 3

之后会进入while循环读取，直到读取到 `KLastType` 或 `KFullType` 的 `record`，或者读到了文件结尾。从日志文件读取到完整的record是 `ReadPhysicalRecord` 完成的，读取出现错误的时候，不会退出，而是汇报错误，继续读取。

```cpp
    // ReadPhysicalRecord may have only had an empty trailer remaining in its
    // internal buffer. Calculate the offset of the next physical record now
    // that it has returned, properly accounting for its header size.
    uint64_t physical_record_offset =
        end_of_buffer_offset_ - buffer_.size() - kHeaderSize - fragment.size();
```

这里存储当前正在读取的record的偏移值，根据不同的record_type，分别处理，共7种情况

#### 3.1 FULL Type

```cpp
case kFullType:
  if (in_fragmented_record) {
    // Handle bug in earlier versions of log::Writer where
    // it could emit an empty kFirstType record at the tail end
    // of a block followed by a kFullType or kFirstType record
    // at the beginning of the next block.
    if (!scratch->empty()) {
      ReportCorruption(scratch->size(), "partial record without end(1)");
    }
  }
  prospective_record_offset = physical_record_offset;
  scratch->clear();
  *record = fragment;
  last_record_offset_ = prospective_record_offset;
  return true;
```