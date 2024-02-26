# SSTable

- [SSTable](#sstable)
  - [Block Format](#block-format)
    - [format](#format)
  - [Write Block](#write-block)
    - [About restartpoint](#about-restartpoint)
    - [Write Interface Requirements](#write-interface-requirements)
  - [Read Block](#read-block)
    - [Read Interface](#read-interface)
    - [Iterator](#iterator)
  - [Build SSTable](#build-sstable)
    - [添加记录](#添加记录)
    - [Flush文件](#flush文件)
    - [`WriteBlock`](#writeblock)
    - [`WriteRawBlock`](#writerawblock)
    - [`Finish`](#finish)
  - [Read SSTable](#read-sstable)
    - [Open](#open)
      - [`ReadBlock()`](#readblock)
      - [`ReadMeta()`](#readmeta)
      - [`ReadFilter()`](#readfilter)
  - [traverse SSTable](#traverse-sstable)
    - [TwoLevelIterator](#twoleveliterator)
      - [`Seek`](#seek)
      - [`BlockReader`](#blockreader)
    - [定位Key](#定位key)
    - [获取Key](#获取key)


> doc/table_format.md

SSTable是Leveldb的核心之一，是表数据最终在磁盘上的物理存储。也是体量比较大的模块。

```shell
    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1]
    ...
    [meta block K]
    [metaindex block]
    [index block]
    [Footer]        (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>
```

- kv有序存储在data block中，block_builder.cc
- meta block存储过滤信息，比如bloom，block_builder.cc
- MetaIndex Block是对Meta Block的索引，它只有一条记录，key是meta index的名字（也就是Filter的名字），value为指向meta index的BlockHandle；BlockHandle是一个结构体，成员offset_是Block在文件中的偏移，成员size_是block的大小；
- Index block是对Data Block的索引，对于其中的每个记录，其key >=Data Block最后一条记录的key，同时<其后Data Block的第一条记录的key；value是指向data index的BlockHandle；
- Footer ： metaindex_handle index_handle padding magic_number(0xdb4775248b80fb57)
  - metaindex_handle ： meta index block的起始
  - index_handle : index block的起始

## Block Format

> table/block_builder.cc table/block_builder.h

### format

每个block由三部分构成。虽然block种类不同，但存储的对象都是有序的kv对，因此管理&读写接口都是统一的。

leveldb对block的管理是读写分离的，读取由`Block`类实现，构建则由`BlockBuilder`实现。

```cpp
unsigned char[] data_;
uint8_t  type_;     // 压缩方式，支持none和snappy
uint32_t crc_;  
```

## Write Block

### About restartpoint

BlockBuilder对key的存储是前缀压缩的，对于有序的字符串来讲，这能极大的减少存储空间。但是却增加了查找的时间复杂度，为了兼顾查找效率，每隔K个key，leveldb就不使用前缀压缩，而是存储整个key，这就是重启点（restartpoint）。

Block在结尾记录所有重启点的偏移，可以二分查找指定的key。Value直接存储在key的后面，无压缩。

```cpp
// include/leveldb/options.h

// Number of keys between restart points for delta encoding of keys.
// This parameter can be changed dynamically.  Most clients should
// leave this parameter alone.
Options::block_restart_interval = 16;
```

对于一个k/v对，其在block中的存储格式为：

- 共享前缀长度 shared_bytes: varint32
- 前缀之后的字符串长度 unshared_bytes: varint32
- 值的长度 value_length: varint32
- 前缀之后的字符串 key_delta: char[unshared_bytes]
- 值 value: char[value_length]

对于重启点，shared_bytes= 0

```cpp
// 该接口要求逐个添加的Key是有序的
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  // 1. 保证新加入的key > 已加入的任何一个key；
  assert(!finished_);
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty()  // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
  size_t shared = 0;
  // 2. 如果计数器counter < opions->block_restart_interval，则使用前缀算法压缩key，否则就把key作为一个重启点，无压缩存储；
  if (counter_ < options_->block_restart_interval) {
    // See how much sharing to do with previous string
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // Restart compression
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }

  // 3. 根据上面的数据格式存储k/v对，追加到buffer中，并更新block状态
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // Add string delta to buffer_ followed by value
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // Update state
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);
  assert(Slice(last_key_) == key);
  counter_++;
}
```

Block的结尾段格式是：

- restarts: uint32[num_restarts]
- num_restarts: uint32 // 重启点个数

```cpp
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}
```

### Write Interface Requirements

```cpp
class BlockBuilder {
  ...

  // Reset the contents as if the BlockBuilder was just constructed.
  // 会直接压入第一个重启点：0
  void Reset();

  // REQUIRES: Finish() has not been called since the last call to Reset().
  // REQUIRES: key is larger than any previously added key
  void Add(const Slice& key, const Slice& value);

  // Finish building the block and return a slice that refers to the
  // block contents.  The returned slice will remain valid for the
  // lifetime of this builder or until Reset() is called.
  Slice Finish();

  // Returns an estimate of the current (uncompressed) size of the block
  // we are building.
  size_t CurrentSizeEstimate() const;

  // Return true iff no entries have been added since the last Reset()
  bool empty() const { return buffer_.empty(); }

 ...
};
```

## Read Block

> table/block.cc  table/block.h

### Read Interface

```cpp
class Block {
    ...

    const char* data_;          // 数据指针
    size_t size_;               // 数据大小
    uint32_t restart_offset_;   // Offset in data_ of restart array
    bool owned_;                // Block owns data_[]

    ...
};
```

```cpp
// 构造主要校验了重启点
Block::Block(const BlockContents& contents)
    : data_(contents.data.data()),
      size_(contents.data.size()),
      owned_(contents.heap_allocated) {
  if (size_ < sizeof(uint32_t)) {
    size_ = 0;  // Error marker
  } else {
    size_t max_restarts_allowed = (size_ - sizeof(uint32_t)) / sizeof(uint32_t);
    if (NumRestarts() > max_restarts_allowed) {
      // The size is too small for NumRestarts()
      size_ = 0;
    } else {
      restart_offset_ = size_ - (1 + NumRestarts()) * sizeof(uint32_t);
    }
  }
}
```

### Iterator

主要包含三个接口：Next Prev Seek

```cpp
// Block的迭代器
class Block::Iter : public Iterator {
 private:
  const Comparator* const comparator_;
  const char* const data_;       // underlying block contents
  uint32_t const restarts_;      // Offset of restart array (list of fixed32)
  uint32_t const num_restarts_;  // Number of uint32_t entries in restart array

  // current_ is offset in data_ of current entry.  >= restarts_ if !Valid
  uint32_t current_;
  uint32_t restart_index_;  // Index of restart block in which current_ falls
  std::string key_;
  Slice value_;
  Status status_;
  ...
};
```

Next: 核心调用，跳到下一个kv对

```cpp 
bool Block::Iter::ParseNextKey() {
    current_ = NextEntryOffset();
    const char* p = data_ + current_;
    const char* limit = data_ + restarts_;  // Restarts come right after data
    if (p >= limit) {
      // No more entries to return.  Mark as invalid.
      current_ = restarts_;
      restart_index_ = num_restarts_;
      return false;
    }

    // Decode next entry
    uint32_t shared, non_shared, value_length;
    p = DecodeEntry(p, limit, &shared, &non_shared, &value_length);
    if (p == nullptr || key_.size() < shared) {
      CorruptionError();
      return false;
    } else {
      key_.resize(shared);
      key_.append(p, non_shared);
      value_ = Slice(p + non_shared, value_length);
      while (restart_index_ + 1 < num_restarts_ &&
             GetRestartPoint(restart_index_ + 1) < current_) {
        ++restart_index_;
      }
      return true;
    }
}

// Helper routine: decode the next block entry starting at "p",
// storing the number of shared key bytes, non_shared key bytes,
// and the length of the value in "*shared", "*non_shared", and
// "*value_length", respectively.  Will not dereference past "limit".
//
// If any errors are detected, returns nullptr.  Otherwise, returns a
// pointer to the key delta (just past the three decoded values).
static inline const char* DecodeEntry(const char* p, const char* limit,
                                      uint32_t* shared, uint32_t* non_shared,
                                      uint32_t* value_length) {
  if (limit - p < 3) return nullptr;
  *shared = reinterpret_cast<const uint8_t*>(p)[0];
  *non_shared = reinterpret_cast<const uint8_t*>(p)[1];
  *value_length = reinterpret_cast<const uint8_t*>(p)[2];
  if ((*shared | *non_shared | *value_length) < 128) {
    // Fast path: all three values are encoded in one byte each
    p += 3;
  } else {
    if ((p = GetVarint32Ptr(p, limit, shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, non_shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, value_length)) == nullptr) return nullptr;
  }

  if (static_cast<uint32_t>(limit - p) < (*non_shared + *value_length)) {
    return nullptr;
  }
  return p;
}
```

Prev: 首先回到current_之前的重启点，然后再向后直到current_

- 二分查找，找到key < target的最后一个重启点
- 找到后，跳转到重启点，其索引由left指定，这是前面二分查找到的结果。如前面所分析的，value_指向重启点的地址，而size_指定为0，这样ParseNextKey函数将会取出重启点的k/v值。

```cpp
void Prev() override {
  assert(Valid());
  // Scan backwards to a restart point before current_
  const uint32_t original = current_;
  while (GetRestartPoint(restart_index_) >= original) {
    // 到第一个entry了，标记invalid状态
    if (restart_index_ == 0) {
      // No more entries
      current_ = restarts_;
      restart_index_ = num_restarts_;
      return;
    }
    restart_index_--;
  }
  //根据restart index定位到重启点的k/v对
  SeekToRestartPoint(restart_index_);
  do {
    // Loop until end of current entry hits the start of original entry
  } while (ParseNextKey() && NextEntryOffset() < original);
}

void SeekToRestartPoint(uint32_t index) {
    key_.clear();
    restart_index_ = index;
    // current_ will be fixed by ParseNextKey();

    // ParseNextKey() starts at the end of value_, so set value_ accordingly
    uint32_t offset = GetRestartPoint(index);
    // 这里的value_并不是k/v对的value，而只是一个指向k/v对起始位置的0长度指针，这样后面的ParseNextKey函数将会取出重启点的k/v值。
    value_ = Slice(data_ + offset, 0);  // value长度设置为0，字符串指针是data_+offset
}
```

Seek 跳到指定的target

```cpp
void Seek(const Slice& target) override {
    // Binary search in restart array to find the last restart point
    // with a key < target
    uint32_t left = 0;
    uint32_t right = num_restarts_ - 1;
    int current_key_compare = 0;

    if (Valid()) {
      // If we're already scanning, use the current position as a starting
      // point. This is beneficial if the key we're seeking to is ahead of the
      // current position.
      current_key_compare = Compare(key_, target);
      if (current_key_compare < 0) {
        // key_ is smaller than target
        left = restart_index_;
      } else if (current_key_compare > 0) {
        right = restart_index_;
      } else {
        // We're seeking to the key we're already at.
        return;
      }
    }

    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;
      uint32_t region_offset = GetRestartPoint(mid);
      uint32_t shared, non_shared, value_length;
      const char* key_ptr =
          DecodeEntry(data_ + region_offset, data_ + restarts_, &shared,
                      &non_shared, &value_length);
      if (key_ptr == nullptr || (shared != 0)) {
        CorruptionError();
        return;
      }
      Slice mid_key(key_ptr, non_shared);
      if (Compare(mid_key, target) < 0) {
        // Key at "mid" is smaller than "target".  Therefore all
        // blocks before "mid" are uninteresting.
        left = mid;
      } else {
        // Key at "mid" is >= "target".  Therefore all blocks at or
        // after "mid" are uninteresting.
        right = mid - 1;
      }
    }

    // We might be able to use our current position within the restart block.
    // This is true if we determined the key we desire is in the current block
    // and is after than the current key.
    assert(current_key_compare == 0 || Valid());
    bool skip_seek = left == restart_index_ && current_key_compare < 0;
    if (!skip_seek) {
      SeekToRestartPoint(left);
    }
    // Linear search (within restart block) for first key >= target
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
  }
```

## Build SSTable

> table/table_builder.cc  include/leveldb/table_builder.h

```cpp
class LEVELDB_EXPORT TableBuilder {
  // Add key,value to the table being constructed.
  // REQUIRES: key is after any previously added key according to comparator.
  // REQUIRES: Finish(), Abandon() have not been called
  void Add(const Slice& key, const Slice& value);

  // Advanced operation: flush any buffered key/value pairs to file.
  // Can be used to ensure that two adjacent entries never live in
  // the same data block.  Most clients should not need to use this method.
  // REQUIRES: Finish(), Abandon() have not been called
  void Flush();

  // Finish building the table.  Stops using the file passed to the
  // constructor after this function returns.
  // REQUIRES: Finish(), Abandon() have not been called
  Status Finish();

  // Indicate that the contents of this builder should be abandoned.  Stops
  // using the file passed to the constructor after this function returns.
  // If the caller is not going to call Finish(), it must call Abandon()
  // before destroying this builder.
  // REQUIRES: Finish(), Abandon() have not been called
  void Abandon();
};
```

类成员定义在 `struct TableBuilder::Rep` 中

```cpp
struct TableBuilder::Rep {
  Options options;              // data block 配置参数
  Options index_block_options;  // index block 配置参数 
  WritableFile* file;           // sstable 文件
  uint64_t offset;              // 写入偏移，起始0
  Status status;                 
  BlockBuilder data_block;      
  BlockBuilder index_block;
  std::string last_key;         // 当前data block的最后一个kv
  int64_t num_entries;          // 当前 data block 的个数
  bool closed;  // Either Finish() or Abandon() has been called.
  FilterBlockBuilder* filter_block; // 可以快速定位key是否在block中
  bool pending_index_entry;         // 初始 false
  BlockHandle pending_handle;       // Handle to add to index block
  std::string compressed_output;    // 压缩后的data block，写出后清空
};
```

### 添加记录

```cpp
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  // 1. 文件保持打开，并且状态OK，否则返回
  assert(!r->closed);   
  if (!ok()) return;
  // 2. 入参 key 必须大于已写入的 last_key
  if (r->num_entries > 0) {
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }
  // 3. 如果标记r->pending_index_entry为true，表明遇到下一个data block的第一个k/v
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    // 根据key调整r->last_key，这是通过Comparator的FindShortestSeparator完成的
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    // 接下来将pending_handle加入到index block中
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }
  // 更新过滤器
  if (r->filter_block != nullptr) {
    r->filter_block->AddKey(key);
  }
  // 更新 last_key
  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  // 如果当前 data block 的大小超限，则立即刷新文件
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

值得讲讲`pending_index_entry`这个标记的意义，见代码注释：

直到遇到下一个databock的第一个key时，我们才为上一个datablock生成index entry，这样的好处是：可以为index使用较短的key；比如上一个data block最后一个k/v的key是"the quick brown fox"，其后继data block的第一个key是"the who"，我们就可以用一个较短的字符串"the r"作为上一个data block的index block entry的key。

简而言之，就是在开始下一个datablock时，Leveldb才将上一个data block加入到index block中。标记pending_index_entry就是干这个用的，对应data block的index entry信息就保存在（BlockHandle）pending_handle。

### Flush文件

```cpp
void TableBuilder::Flush() {
  Rep* r = rep_;
  // 首先保证未关闭，且状态ok
  assert(!r->closed);
  if (!ok()) return;
  // 若 datablock 是空的，直接返回
  if (r->data_block.empty()) return;
  // 保证pending_index_entry为false，即data block的Add已经完成
  assert(!r->pending_index_entry);
  // 写入data block，并设置其index entry信息—BlockHandle对象
  WriteBlock(&r->data_block, &r->pending_handle);
  //写入成功，则Flush文件，并设置r->pending_index_entry为true，
  //以根据下一个data block的first key调整index entry的key—即r->last_key
  if (ok()) {
    r->pending_index_entry = true;
    r->status = r->file->Flush();
  }
  if (r->filter_block != nullptr) {
    //将data block在sstable中的偏移加入到filter block中
    r->filter_block->StartBlock(r->offset);
    // 并指明开始新的data block
  }
}
```

### `WriteBlock`

```cpp
void TableBuilder::WriteBlock(BlockBuilder* block, BlockHandle* handle) {
  // File format contains a sequence of blocks where each block has:
  //    block_data: uint8[n]
  //    type: uint8
  //    crc: uint32
  assert(ok());
  Rep* r = rep_;
  Slice raw = block->Finish();
  // 获得data block的序列化字符串
  Slice block_contents;
  CompressionType type = r->options.compression;
  // TODO(postrelease): Support more compression options: zlib?
  switch (type) {
    case kNoCompression:
      block_contents = raw;
      break;

    case kSnappyCompression: {
      ...
      break;
    }

    case kZstdCompression: {
      ...
      break;
    }
  }
  // 将data内容写入到文件，并重置block成初始化状态，清空compressedoutput
  WriteRawBlock(block_contents, type, handle);
  r->compressed_output.clear();
  block->Reset();
}
```

### `WriteRawBlock`

```cpp
void TableBuilder::WriteRawBlock(const Slice& block_contents,
                                 CompressionType type, BlockHandle* handle) {
  Rep* r = rep_;
  // 更新偏移信息
  handle->set_offset(r->offset);
  handle->set_size(block_contents.size());
  // 写入文件
  r->status = r->file->Append(block_contents);
  if (r->status.ok()) {
    // 写入 1 byte 类型 和 4 byte crc
    char trailer[kBlockTrailerSize];
    trailer[0] = type;
    uint32_t crc = crc32c::Value(block_contents.data(), block_contents.size());
    crc = crc32c::Extend(crc, trailer, 1);  // Extend crc to cover block type
    EncodeFixed32(trailer + 1, crc32c::Mask(crc));
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      // 写入成功之后，更新下一个data-block的写入位置
      r->offset += block_contents.size() + kBlockTrailerSize;
    }
  }
}
```

### `Finish`

调用Finish函数，表明调用者将所有已经添加的k/v对持久化到sstable，并关闭sstable文件。

```cpp
Status TableBuilder::Finish() {
  Rep* r = rep_;
  // 先写入最后一块datablock
  Flush();
  assert(!r->closed);
  r->closed = true; // 标志置位，不允许再添加kv

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write filter block
  if (ok() && r->filter_block != nullptr) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != nullptr) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write index block
  if (ok()) {
    if (r->pending_index_entry) { 
      // 成功Flush过data block，那么需要为最后一块data block设置index block，并加入到index block中
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

## Read SSTable

> include/leveldb/table.h table/table.cc

```cpp
// A Table is a sorted map from strings to strings.  Tables are
// immutable and persistent.  A Table may be safely accessed from
// multiple threads without external synchronization.
class LEVELDB_EXPORT Table 
```

### Open

```cpp
// Attempt to open the table that is stored in bytes [0..file_size)
// of "file", and read the metadata entries necessary to allow
// retrieving data from the table.
//
// If successful, returns ok and sets "*table" to the newly opened
// table.  The client should delete "*table" when no longer needed.
// If there was an error while initializing the table, sets "*table"
// to nullptr and returns a non-ok status.  Does not take ownership of
// "*source", but the client must ensure that "source" remains live
// for the duration of the returned table's lifetime.
//
// *file must remain live while this Table is in use.
Status Table::Open(const Options& options, RandomAccessFile* file,
                   uint64_t size, Table** table) {
  *table = nullptr;
  if (size < Footer::kEncodedLength) {    // 文件太短，报错误
    return Status::Corruption("file is too short to be an sstable");
  }
  // 先读Footer
  char footer_space[Footer::kEncodedLength];
  Slice footer_input;
  Status s = file->Read(size - Footer::kEncodedLength, Footer::kEncodedLength,
                        &footer_input, footer_space);
  if (!s.ok()) return s;

  Footer footer;
  s = footer.DecodeFrom(&footer_input);
  if (!s.ok()) return s;

  // Read the index block
  BlockContents index_block_contents;
  ReadOptions opt;
  if (options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  s = ReadBlock(file, opt, footer.index_handle(), &index_block_contents);

  if (s.ok()) {
    // We've successfully read the footer and the index block: we're
    // ready to serve requests.
    Block* index_block = new Block(index_block_contents);
    Rep* rep = new Table::Rep;  
    rep->options = options;
    rep->file = file;
    rep->metaindex_handle = footer.metaindex_handle();
    rep->index_block = index_block;
    rep->cache_id = (options.block_cache ? options.block_cache->NewId() : 0); // 如果option打开了cache，还要为table创建cache
    rep->filter = nullptr;
    *table = new Table(rep);  // 构建table对象
    (*table)->ReadMeta(footer); // 读取metaindex数据构建filter policy
  }

  return s;
}
```

#### `ReadBlock()`

```cpp
// table/format.cc
Status ReadBlock(RandomAccessFile* file, const ReadOptions& options,
                 const BlockHandle& handle, BlockContents* result) {
  // 初始化结果result
  result->data = Slice();
  result->cachable = false;
  result->heap_allocated = false;

  // Read the block contents as well as the type/crc footer.
  // See table_builder.cc for the code that built this structure.
  size_t n = static_cast<size_t>(handle.size());
  char* buf = new char[n + kBlockTrailerSize];  // kBlockTrailerSize=5 : 1-byte type + 32-bit crc
  Slice contents;
  Status s = file->Read(handle.offset(), n + kBlockTrailerSize, &contents, buf);
  if (!s.ok()) {
    delete[] buf;
    return s;
  }
  if (contents.size() != n + kBlockTrailerSize) {
    delete[] buf;
    return Status::Corruption("truncated block read");
  }

  // Check the crc of the type and the block contents
  const char* data = contents.data();  // Pointer to where Read put the data
  // 如果要求校验crc
  if (options.verify_checksums) {
    const uint32_t crc = crc32c::Unmask(DecodeFixed32(data + n + 1));
    const uint32_t actual = crc32c::Value(data, n + 1);
    if (actual != crc) {
      delete[] buf;
      s = Status::Corruption("block checksum mismatch");
      return s;
    }
  }
  // 根据type指定的存储类型，如果是非压缩的，则直接取数据赋给result，否则先解压，把解压结果赋给result
  switch (data[n]) {
    case kNoCompression:
      ...
      // Ok
      break;
    case kSnappyCompression: {
      ...
      break;
    }
    case kZstdCompression: {
      ...
      break;
    }
    default:
      delete[] buf;
      return Status::Corruption("bad block type");
  }

  return Status::OK();
}
```

#### `ReadMeta()`

```cpp
void Table::ReadMeta(const Footer& footer) {
  // 不需要metadata
  if (rep_->options.filter_policy == nullptr) {
    return;  // Do not need any metadata
  }

  // TODO(sanjay): Skip this if footer.metaindex_handle() size indicates
  // it is an empty block.
  ReadOptions opt;
  if (rep_->options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  // 根据读取的content构建Block
  BlockContents contents;
  if (!ReadBlock(rep_->file, opt, footer.metaindex_handle(), &contents).ok()) {
    // Do not propagate errors since meta info is not needed for operation
    return;
  }
  Block* meta = new Block(contents);
  // 找到指定的filter
  Iterator* iter = meta->NewIterator(BytewiseComparator());
  std::string key = "filter.";
  key.append(rep_->options.filter_policy->Name());
  iter->Seek(key);
  if (iter->Valid() && iter->key() == Slice(key)) { // 如果找到了就调用ReadFilter构建filter对象
    ReadFilter(iter->value());
  }
  delete iter;
  delete meta;
}
```

#### `ReadFilter()`

```cpp
void Table::ReadFilter(const Slice& filter_handle_value) {
  // 从传入的filter_handle_value Decode出BlockHandle，这是filter的偏移和大小
  Slice v = filter_handle_value;
  BlockHandle filter_handle;
  if (!filter_handle.DecodeFrom(&v).ok()) {
    return;
  }

  // We might want to unify with ReadBlock() if we start
  // requiring checksum verification in Table::Open.
  ReadOptions opt;
  if (rep_->options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  BlockContents block;
  if (!ReadBlock(rep_->file, opt, filter_handle, &block).ok()) {
    return;
  }
  // 如果block的heap_allocated为true，表明需要自行释放内存，因此要把指针保存在filter_data中
  if (block.heap_allocated) {
    rep_->filter_data = block.data.data();  // Will need to delete later
  }
  // 根据读取的data创建FilterBlockReader对象
  rep_->filter = new FilterBlockReader(rep_->options.filter_policy, block.data);
}
```

## traverse SSTable

> Table导出了一个返回Iterator的接口，通过Iterator对象，调用者就可以遍历Table的内容，它简单的返回了一个TwoLevelIterator对象

```cpp
// Returns a new iterator over the table contents.
// The result of NewIterator() is initially invalid (caller must
// call one of the Seek methods on the iterator before using it).
Iterator* Table::NewIterator(const ReadOptions& options) const {
  return NewTwoLevelIterator(
      rep_->index_block->NewIterator(rep_->options.comparator),
      &Table::BlockReader, const_cast<Table*>(this), options);
}
```

### TwoLevelIterator

> table/two_level_iterator.cc

它也是Iterator的子类，之所以叫two level应该是不仅可以迭代其中存储的对象，它还接受了一个函数BlockFunction，可以遍历存储的对象，可见它是专门为Table定制的。 我们已经知道各种Block的存储格式都是相同的，但是各自block data存储的k/v又互不相同，于是我们就需要一个途径，能够在使用同一个方式遍历不同的block时，又能解析这些k/v。这就是BlockFunction，它又返回了一个针对block data的Iterator。Block和block data存储的k/v对的key是统一的。

```cpp
// Return a new two level iterator.  A two-level iterator contains an
// index iterator whose values point to a sequence of blocks where
// each block is itself a sequence of key,value pairs.  The returned
// two-level iterator yields the concatenation of all key/value pairs
// in the sequence of blocks.  Takes ownership of "index_iter" and
// will delete it when no longer needed.
//
// Uses a supplied function to convert an index_iter value into
// an iterator over the contents of the corresponding block.
Iterator* NewTwoLevelIterator(
    Iterator* index_iter,
    Iterator* (*block_function)(void* arg, const ReadOptions& options,
                                const Slice& index_value),
    void* arg, const ReadOptions& options);

  
class TwoLevelIterator : public Iterator {
  BlockFunction block_function_;
  void* arg_;
  const ReadOptions options_;
  Status status_;
  IteratorWrapper index_iter_;  // 遍历block的迭代器
  IteratorWrapper data_iter_;  // May be nullptr，遍历block data的迭代器
  // If data_iter_ is non-null, then "data_block_handle_" holds the
  // "index_value" passed to block_function_ to create the data_iter_.
  std::string data_block_handle_;
};

// A internal wrapper class with an interface similar to Iterator that
// caches the valid() and key() results for an underlying iterator.
// This can help avoid virtual function calls and also gives better
// cache locality.
// table/iterator_wrapper.h
class IteratorWrapper {};
```

#### `Seek`

除了`Seek`，还提供了`SeekToFirst` `SeekToLast` `Next` `Prev`

```cpp
void TwoLevelIterator::Seek(const Slice& target) {
  index_iter_.Seek(target);   // 查找block的位置
  InitDataBlock();            // 初始化block的iter
  if (data_iter_.iter() != nullptr) data_iter_.Seek(target);  // 在block中查找
  SkipEmptyDataBlocksForward(); // 向前跳过空白的block
}

void TwoLevelIterator::InitDataBlock() {
  if (!index_iter_.Valid()) {   // index_iter非法时，给个nullptr
    SetDataIterator(nullptr); 
  } else {
    Slice handle = index_iter_.value();
    if (data_iter_.iter() != nullptr &&
        handle.compare(data_block_handle_) == 0) {  
      // data_iter_ is already constructed with this iterator, so
      // no need to change anything
    } else {
      Iterator* iter = (*block_function_)(arg_, options_, handle);
      data_block_handle_.assign(handle.data(), handle.size());
      SetDataIterator(iter);
    }
  }
}

void TwoLevelIterator::SkipEmptyDataBlocksForward() {
  while (data_iter_.iter() == nullptr || !data_iter_.Valid()) {
    // Move to next block
    if (!index_iter_.Valid()) {
      SetDataIterator(nullptr);
      return;
    }
    index_iter_.Next();
    InitDataBlock();
    if (data_iter_.iter() != nullptr) data_iter_.SeekToFirst();
  }
}
```

#### `BlockReader`

> table/table.cc

如下可见，构建迭代器的时候，传入的 BlockFunc 是 `Table::BlockReader`

```cpp
Iterator* Table::NewIterator(const ReadOptions& options) const {
  return NewTwoLevelIterator(
      rep_->index_block->NewIterator(rep_->options.comparator),
      &Table::BlockReader, const_cast<Table*>(this), options);
}
```

```cpp
// Convert an index iterator value (i.e., an encoded BlockHandle)
// into an iterator over the contents of the corresponding block.
Iterator* Table::BlockReader(void* arg, const ReadOptions& options,
                             const Slice& index_value) {
  Table* table = reinterpret_cast<Table*>(arg);
  Cache* block_cache = table->rep_->options.block_cache;
  Block* block = nullptr;
  Cache::Handle* cache_handle = nullptr;

  BlockHandle handle;
  Slice input = index_value;
  Status s = handle.DecodeFrom(&input); // 从参数中解析出Handle对象，读取Block索引
  // We intentionally allow extra stuff in index_value so that we
  // can add more features in the future.

  if (s.ok()) {
    BlockContents contents;
    if (block_cache != nullptr) {
      char cache_key_buffer[16];
      // cache key的格式是：cache id + offset
      EncodeFixed64(cache_key_buffer, table->rep_->cache_id);
      EncodeFixed64(cache_key_buffer + 8, handle.offset());
      Slice key(cache_key_buffer, sizeof(cache_key_buffer));
      cache_handle = block_cache->Lookup(key);
      if (cache_handle != nullptr) {    // 优先从缓存中获取，没命中的情况下再去读文件
        // cache存在则直接读值
        block = reinterpret_cast<Block*>(block_cache->Value(cache_handle));
      } else {
        s = ReadBlock(table->rep_->file, options, handle, &contents);
        if (s.ok()) {
          block = new Block(contents);
          if (contents.cachable && options.fill_cache) {
            // 插入时同步注册了释放的函数
            cache_handle = block_cache->Insert(key, block, block->size(),
                                               &DeleteCachedBlock);
          }
        }
      }
    } else {
      s = ReadBlock(table->rep_->file, options, handle, &contents);
      if (s.ok()) {
        block = new Block(contents);
      }
    }
  }

  Iterator* iter;
  if (block != nullptr) {
    iter = block->NewIterator(table->rep_->options.comparator);
    if (cache_handle == nullptr) {
      iter->RegisterCleanup(&DeleteBlock, block, nullptr);
    } else {
      iter->RegisterCleanup(&ReleaseBlock, block_cache, cache_handle);
    }
  } else {
    iter = NewErrorIterator(s);
  }
  return iter;
}
```

### 定位Key

并非精确定位，而是在table中找到第一个 >= 指定key的kv对，然后返回其value在sstable文件中的偏移

```cpp
  // Given a key, return an approximate byte offset in the file where
  // the data for that key begins (or would begin if the key were
  // present in the file).  The returned value is in terms of file
  // bytes, and so includes effects like compression of the underlying data.
  // E.g., the approximate offset of the last key in the table will
  // be close to the file length.
uint64_t Table::ApproximateOffsetOf(const Slice& key) const {
  Iterator* index_iter =
      rep_->index_block->NewIterator(rep_->options.comparator);
  // 调用Block::Iter的Seek函数定位
  index_iter->Seek(key);
  uint64_t result;
  if (index_iter->Valid()) {
    BlockHandle handle;
    Slice input = index_iter->value();
    Status s = handle.DecodeFrom(&input);
    if (s.ok()) {
      result = handle.offset(); // 如果index_iter是合法的值，并且Decode成功，返回结果offset
    } else {
      // Strange: we can't decode the block handle in the index block.
      // We'll just return the offset of the metaindex block, which is
      // close to the whole file size for this case.
      result = rep_->metaindex_handle.offset(); // 其它情况，设置result为rep_->metaindex_handle.offset()，metaindex的偏移在文件结尾附近
    }
  } else {
    // key is past the last key in the file.  Approximate the offset
    // by returning the offset of the metaindex block (which is
    // right near the end of the file).
    result = rep_->metaindex_handle.offset();
  }
  delete index_iter;
  return result;
}
```

### 获取Key

仅被table_cache使用

```cpp
// Calls (*handle_result)(arg, ...) with the entry found after a call
// to Seek(key).  May not make such a call if filter policy says
// that key is not present.
Status Table::InternalGet(const ReadOptions& options, const Slice& k, void* arg,
                          void (*handle_result)(void*, const Slice&,
                                                const Slice&)) {
  Status s;
  Iterator* iiter = rep_->index_block->NewIterator(rep_->options.comparator);
  iiter->Seek(k);
  if (iiter->Valid()) {
    Slice handle_value = iiter->value();
    FilterBlockReader* filter = rep_->filter;
    BlockHandle handle;
    if (filter != nullptr && handle.DecodeFrom(&handle_value).ok() &&
        !filter->KeyMayMatch(handle.offset(), k)) {
      // Not found
    } else {
      Iterator* block_iter = BlockReader(this, options, iiter->value());
      block_iter->Seek(k);
      if (block_iter->Valid()) {
        (*handle_result)(arg, block_iter->key(), block_iter->value());
      }
      s = block_iter->status();
      delete block_iter;
    }
  }
  if (s.ok()) {
    s = iiter->status();
  }
  delete iiter;
  return s;
}
```