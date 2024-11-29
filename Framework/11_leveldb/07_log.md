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

```cpp
static const int kBlockSize = 32768;

// Header is checksum (4 bytes), length (2 bytes), type (1 byte).
static const int kHeaderSize = 4 + 2 + 1;
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
bool Reader::ReadRecord(Slice* record, std::string* scratch) {
  // S1 根据initial offset跳转到调用者指定的位置，开始读取日志文件。跳转就是直接调用SequentialFile的Seek接口。
  if (last_record_offset_ < initial_offset_) {  // 当前偏移 < 指定的偏移，需要Seek
    if (!SkipToInitialBlock()) {
      return false;
    }
  }
  // S2 在开始while循环前首先初始化几个标记：
  scratch->clear();
  record->clear();
  bool in_fragmented_record = false;
  // Record offset of the logical record that we're reading
  // 0 is a dummy value to make compilers happy
  uint64_t prospective_record_offset = 0;

  // S3 进入到while(true)循环，直到读取到KLastType或者KFullType的record，或者到了文件结尾。从日志文件读取完整的record是ReadPhysicalRecord函数完成的。
  Slice fragment;
  while (true) {
    // S3.1 从文件读取record
    const unsigned int record_type = ReadPhysicalRecord(&fragment);

    // ReadPhysicalRecord may have only had an empty trailer remaining in its
    // internal buffer. Calculate the offset of the next physical record now
    // that it has returned, properly accounting for its header size.
    uint64_t physical_record_offset =
        end_of_buffer_offset_ - buffer_.size() - kHeaderSize - fragment.size();

    if (resyncing_) {
      if (record_type == kMiddleType) {
        continue;
      } else if (record_type == kLastType) {
        resyncing_ = false;
        continue;
      } else {
        resyncing_ = false;
      }
    }

    switch (record_type) {
      case kFullType:   // 表明是一条完整的log record，成功返回读取的user record数据。另外需要对早期版本做些work around，早期的Leveldb会在block的结尾生产一条空的kFirstType log record。
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

      case kFirstType:  // 把数据读取到scratch中，直到成功读取了LAST类型的log record，才把数据返回到result中，继续下次的读取循环。
        if (in_fragmented_record) {
          // Handle bug in earlier versions of log::Writer where
          // it could emit an empty kFirstType record at the tail end
          // of a block followed by a kFullType or kFirstType record
          // at the beginning of the next block.
          if (!scratch->empty()) {  // 如果再次遇到FIRST or FULL类型的log record，如果scratch不为空，就说明日志文件有错误。
            ReportCorruption(scratch->size(), "partial record without end(2)");
          }
        }
        prospective_record_offset = physical_record_offset;
        scratch->assign(fragment.data(), fragment.size());
        in_fragmented_record = true;
        break;

      case kMiddleType:
        if (!in_fragmented_record) {
          // 如果不是在fragment中，报告错误，否则直接append到scratch中就可以了。
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record(1)");
        } else {
          scratch->append(fragment.data(), fragment.size());
        }
        break;

      case kLastType:
        // 说明是一系列log record(fragment)中的最后一条。如果不在fragment中，报告错误。
        if (!in_fragmented_record) {
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record(2)");
        } else {
          scratch->append(fragment.data(), fragment.size());
          *record = Slice(*scratch);
          last_record_offset_ = prospective_record_offset;
          return true;
        }
        break;

      case kEof:  // 扩展类型，遇到文件结尾，直接false，不返回任何结果
        if (in_fragmented_record) {
          // This can be caused by the writer dying immediately after
          // writing a physical record but before completing the next; don't
          // treat it as a corruption, just ignore the entire logical record.
          scratch->clear();
        }
        return false;

      // 非法的record，当前有3中情况会返回bad record：
      // * CRC校验失败 (ReadPhysicalRecord reports adrop)
      // * 长度为0 (No drop is reported)
      // * 在指定的initial_offset之外 (No drop is reported)
      case kBadRecord:
        if (in_fragmented_record) {
          ReportCorruption(scratch->size(), "error in middle of record");
          in_fragmented_record = false;
          scratch->clear();
        }
        break;

      default: {
        char buf[40];
        std::snprintf(buf, sizeof(buf), "unknown record type %u", record_type);
        ReportCorruption(
            (fragment.size() + (in_fragmented_record ? scratch->size() : 0)),
            buf);
        in_fragmented_record = false;
        scratch->clear();
        break;
      }
    }
  }
  return false;
}
```

### `ReadPhysicalRecord`

```cpp
unsigned int Reader::ReadPhysicalRecord(Slice* result) {
  while (true) {  // 入口死循环，目的是为了读取到一个完整的record
    // 1. 如果buffer_小于block header大小kHeaderSize，进入如下的几个分支：
    if (buffer_.size() < kHeaderSize) {
      // 1.1 不是文件结尾，则清空buffer，读取数据
      if (!eof_) {
        // Last read was a full read, so this is a trailer to skip
        buffer_.clear();
        Status status = file_->Read(kBlockSize, &buffer_, backing_store_);
        end_of_buffer_offset_ += buffer_.size();
        if (!status.ok()) { 
          // 读取失败则情况buf，并报错，返回kEof
          buffer_.clear();
          ReportDrop(kBlockSize, status);
          eof_ = true;
          return kEof;
        } else if (buffer_.size() < kBlockSize) { 
          // 实际读取字节不足，说明读取到了文件尾部，置eof
          eof_ = true;
        }
        continue;
      // 1.2 如果是文件结尾，buf非空；则清空buf，直接返回；兼容了写错误，隐含了buf为空，也是这个分支返回
      } else {
        // Note that if buffer_ is non-empty, we have a truncated header at the
        // end of the file, which can be caused by the writer crashing in the
        // middle of writing the header. Instead of considering this an error,
        // just report EOF.
        buffer_.clear();
        return kEof;
      }
    }
    // 2. 进入到这里表明上次循环中的Read读取到了一个完整的log record，continue后的第二次循环判断buffer_.size() >= kHeaderSize将执行到此处。
    // Parse the header
    const char* header = buffer_.data();
    const uint32_t a = static_cast<uint32_t>(header[4]) & 0xff;
    const uint32_t b = static_cast<uint32_t>(header[5]) & 0xff;
    const unsigned int type = header[6];
    const uint32_t length = a | (b << 8);
    if (kHeaderSize + length > buffer_.size()) {
      // 如果长度校验失败，则返回错误
      size_t drop_size = buffer_.size();
      buffer_.clear();
      if (!eof_) {
        ReportCorruption(drop_size, "bad record length");
        return kBadRecord;
      }
      // If the end of the file has been reached without reading |length| bytes
      // of payload, assume the writer died in the middle of writing the record.
      // Don't report a corruption.
      return kEof;
    }
    // 特殊类型，不汇报错误
    if (type == kZeroType && length == 0) {
      // Skip zero length record without reporting any drops since
      // such records are produced by the mmap based writing code in
      // env_posix.cc that preallocates file regions.
      buffer_.clear();
      return kBadRecord;
    }

    // 3. crc校验
    if (checksum_) {
      uint32_t expected_crc = crc32c::Unmask(DecodeFixed32(header));
      uint32_t actual_crc = crc32c::Value(header + 6, 1 + length);
      if (actual_crc != expected_crc) {
        // Drop the rest of the buffer since "length" itself may have
        // been corrupted and if we trust it, we could find some
        // fragment of a real log record that just happens to look
        // like a valid log record.
        size_t drop_size = buffer_.size();
        buffer_.clear();
        ReportCorruption(drop_size, "checksum mismatch");
        return kBadRecord;
      }
    }
    
    buffer_.remove_prefix(kHeaderSize + length);
    // 如果record的开始位置在initial offset之前，则跳过，并返回kBadRecord，否则返回record数据和type。
    // Skip physical record that started before initial_offset_
    if (end_of_buffer_offset_ - buffer_.size() - kHeaderSize - length <
        initial_offset_) {
      result->clear();
      return kBadRecord;
    }

    *result = Slice(header + kHeaderSize, length);
    return type;
  }
}
```