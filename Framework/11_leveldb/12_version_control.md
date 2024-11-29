# Version Control

- [Version Control](#version-control)
  - [VersionSet \& Version](#versionset--version)
  - [manifest文件格式](#manifest文件格式)
  - [Version 接口](#version-接口)


**版本（Version）**:

- 每个版本代表了一个数据库状态的快照，包括当前磁盘和内存中的所有文件信息。
- 版本集（VersionSet）是所有版本的集合，负责版本间的管理和切换。
- 当前版本（CURRENT）是活跃的版本，所有的读写操作都在这个版本上进行。
- 执行压缩（compaction）操作后，LevelDB会基于当前版本创建一个新的版本，当前版本随之成为历史版本。

**迭代器与版本依赖**:

- 创建的迭代器（Iterator）将依附于创建时的版本。即使该版本后来变成了历史版本，迭代器仍然可以访问它，因为这个版本不会删除。

**版本编辑（VersionEdit）**:

- 版本编辑记录了版本之间的变化，可以看作是版本更新的增量（delta）。
- 版本编辑记录了新增和删除的文件，以及文件之间的合并操作。
- 通过应用版本编辑到当前版本上，可以生成一个新的版本。这个过程可以表示为：Version0 + VersionEdit --> Version1。

**MANIFEST文件**:

- MANIFEST文件是LevelDB的元数据文件，记录了当前版本的所有信息。
- 每当版本发生变化时，新的版本信息就会保存到MANIFEST文件中。
- MANIFEST文件以版本编辑的形式组织，采用日志文件格式，使用日志的读写方式进行操作。
- 每个版本编辑都是一条日志记录，这样MANIFEST文件本身就是一个变更日志。

## VersionSet & Version

> db/version_set.h  db/version_edit.h

```cpp
class Version {
  VersionSet* vset_;  // VersionSet to which this Version belongs
  Version* next_;     // Next version in linked list
  Version* prev_;     // Previous version in linked list
  int refs_;          // Number of live refs to this version

  // List of files per level
  std::vector<FileMetaData*> files_[config::kNumLevels];

  // Next file to compact based on seek stats.
  FileMetaData* file_to_compact_;
  int file_to_compact_level_;

  // Level that should be compacted next and its compaction score.
  // Score < 1 means compaction is not strictly needed.  These fields
  // are initialized by Finalize().
  double compaction_score_;
  int compaction_level_;
};
```

一个Version就是一个sstable文件集合，以及它管理的compact状态。

```cpp
class VersionSet {
  // 第一组，直接来自于DBImple，构造函数传入
  Env* const env_;
  const std::string dbname_;
  const Options* const options_;
  TableCache* const table_cache_;
  const InternalKeyComparator icmp_;

  // 第二组，db元信息相关
  uint64_t next_file_number_;
  uint64_t manifest_file_number_;
  uint64_t last_sequence_;
  uint64_t log_number_;
  uint64_t prev_log_number_;  // 0 or backing store for memtable being compacted

  // Opened lazily, 第三组，menifest文件相关 
  WritableFile* descriptor_file_;
  log::Writer* descriptor_log_;

  // 第四组，版本管理 
  Version dummy_versions_;  // Head of circular doubly-linked list of versions.
  Version* current_;        // == dummy_versions_.prev_

  // Per-level key at which the next compaction at that level should start.
  // Either an empty string, or a valid InternalKey.
  std::string compact_pointer_[config::kNumLevels];
};
```

```cpp
class VersionEdit {
    // 成员变量，由此可大概窥得DB元信息的内容。  
    typedef std::set< std::pair<int, uint64_t> > DeletedFileSet;  
    std::string comparator_; // key comparator名字  
    uint64_t log_number_; // 日志编号  
    uint64_t prev_log_number_; // 前一个日志编号  
    uint64_t next_file_number_; // 下一个文件编号  
    SequenceNumber last_sequence_; // 上一个seq  
    bool has_comparator_; // 是否有comparator  
    bool has_log_number_;// 是否有log_number_  
    bool has_prev_log_number_;// 是否有prev_log_number_  
    bool has_next_file_number_;// 是否有next_file_number_  
    bool has_last_sequence_;// 是否有last_sequence_  
    std::vector< std::pair<int, InternalKey> >compact_pointers_; // compact点  
    DeletedFileSet deleted_files_; // 删除文件集合  
    std::vector< std::pair<int, FileMetaData> > new_files_; // 新文件集合 
};
```

## manifest文件格式

通过 `VersionEdit::EncodeTo` 接口可以得知 `manifest` 中，每一条记录的格式。

首先是使用的coparator名、log编号、前一个log编号、下一个文件编号、上一个序列号。这些都是日志、sstable文件使用到的重要信息，这些字段不一定必然存在。 Leveldb在写入每个字段之前，都会先写入一个varint型数字来标记后面的字段类型。在读取时，先读取此字段，根据类型解析后面的信息。一共有9种类型：

```cpp
// Tag numbers for serialized VersionEdit.  These numbers are written to
// disk and should not be changed.
enum Tag {
  kComparator = 1,
  kLogNumber = 2,
  kNextFileNumber = 3,
  kLastSequence = 4,
  kCompactPointer = 5,
  kDeletedFile = 6,
  kNewFile = 7,
  // 8 was used for large value refs
  kPrevLogNumber = 9
};
```

数字都是varint存储格式，string都是以varint指明其长度，后面跟实际的字符串内容。

```cpp

void VersionEdit::EncodeTo(std::string* dst) const {
  if (has_comparator_) {
    PutVarint32(dst, kComparator);
    PutLengthPrefixedSlice(dst, comparator_);
  }
  if (has_log_number_) {
    PutVarint32(dst, kLogNumber);
    PutVarint64(dst, log_number_);
  }
  if (has_prev_log_number_) {
    PutVarint32(dst, kPrevLogNumber);
    PutVarint64(dst, prev_log_number_);
  }
  if (has_next_file_number_) {
    PutVarint32(dst, kNextFileNumber);
    PutVarint64(dst, next_file_number_);
  }
  if (has_last_sequence_) {
    PutVarint32(dst, kLastSequence);
    PutVarint64(dst, last_sequence_);
  }

  for (size_t i = 0; i < compact_pointers_.size(); i++) {
    PutVarint32(dst, kCompactPointer);
    PutVarint32(dst, compact_pointers_[i].first);  // level
    PutLengthPrefixedSlice(dst, compact_pointers_[i].second.Encode());
  }

  for (const auto& deleted_file_kvp : deleted_files_) {
    PutVarint32(dst, kDeletedFile);
    PutVarint32(dst, deleted_file_kvp.first);   // level
    PutVarint64(dst, deleted_file_kvp.second);  // file number
  }

  for (size_t i = 0; i < new_files_.size(); i++) {
    const FileMetaData& f = new_files_[i].second;
    PutVarint32(dst, kNewFile);
    PutVarint32(dst, new_files_[i].first);  // level
    PutVarint64(dst, f.number);
    PutVarint64(dst, f.file_size);
    PutLengthPrefixedSlice(dst, f.smallest.Encode());
    PutLengthPrefixedSlice(dst, f.largest.Encode());
  }
}
```

## Version 接口

```c
  // Append to *iters a sequence of iterators that will
  // yield the contents of this Version when merged together.
  // REQUIRES: This version has been saved (see VersionSet::SaveTo)
  void AddIterators(const ReadOptions&, std::vector<Iterator*>* iters);

  // Lookup the value for key.  If found, store it in *val and
  // return OK.  Else return a non-OK status.  Fills *stats.
  // REQUIRES: lock is not held
  Status Get(const ReadOptions&, const LookupKey& key, std::string* val,
             GetStats* stats);
```