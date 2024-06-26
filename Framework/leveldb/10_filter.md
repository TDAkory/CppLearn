# Filter

> include/leveldb/filter_policy.h table/filter_block.h

- [Filter](#filter)
  - [`FilterPolicy`接口](#filterpolicy接口)
  - [`BloomFilterPolicy`](#bloomfilterpolicy)
  - ["filter" Meta Block](#filter-meta-block)
  - [创建`FilterBlock`](#创建filterblock)
  - [读取`FilterBlock`](#读取filterblock)

## `FilterPolicy`接口

注意其 `KeyMayMatch` 接口并不是精确的，入参不在 `CreateFilter` 接口创造的列表中时，仅要求有较高的概率返回`false`。

```cpp
class LEVELDB_EXPORT FilterPolicy {
 public:
  virtual ~FilterPolicy();

  // Return the name of this policy.  Note that if the filter encoding
  // changes in an incompatible way, the name returned by this method
  // must be changed.  Otherwise, old incompatible filters may be
  // passed to methods of this type.
  virtual const char* Name() const = 0;

  // keys[0,n-1] contains a list of keys (potentially with duplicates)
  // that are ordered according to the user supplied comparator.
  // Append a filter that summarizes keys[0,n-1] to *dst.
  //
  // Warning: do not change the initial contents of *dst.  Instead,
  // append the newly constructed filter to *dst.
  virtual void CreateFilter(const Slice* keys, int n,
                            std::string* dst) const = 0;

  // "filter" contains the data appended by a preceding call to
  // CreateFilter() on this class.  This method must return true if
  // the key was in the list of keys passed to CreateFilter().
  // This method may return true or false if the key was not on the
  // list, but it should aim to return false with a high probability.
  virtual bool KeyMayMatch(const Slice& key, const Slice& filter) const = 0;
};
```

包含两个派生：

* `InternalFilterPolicy`：wrapper，把FilterPolicy应用在InternalKey上。其思路是将InternalKey拆分得到user key，然后在user key上做FilterPolicy的操作。
* `BloomFilterPolicy`：如下

## `BloomFilterPolicy`

Leveldb使用了double hashing来模拟多个hash函数，Double hashing从一个hash值开始，重复向前迭代，直到解决冲突或者搜索完hash表。不同的是，double hashing使用的是另外一个hash函数，而不是固定的步长。 给定两个独立的hash函数h1和h2，对于hash表T和值k，第i次迭代计算出的位置就是：h(i, k) = (h1(k) + i*h2(k)) mod |T|。 对此，Leveldb选择的hash函数是：

```shell
Gi(x)=H1(x)+iH2(x)                  # H1基本hash函数，H2由Hq循环右移得到，i是循环次数
H2(x)=(H1(x)>>17) | (H1(x)<<15)
```

```cpp
static uint32_t BloomHash(const Slice& key) {
  return Hash(key.data(), key.size(), 0xbc9f1d34);
}

class BloomFilterPolicy : public FilterPolicy {
 public:
  explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }

  const char* Name() const override { return "leveldb.BuiltinBloomFilter2"; }

  void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
    // Compute bloom filter size (in both bits and bytes)
    size_t bits = n * bits_per_key_;

    // For small n, we can see a very high false positive rate.  Fix it
    // by enforcing a minimum bloom filter length.
    if (bits < 64) bits = 64;

    size_t bytes = (bits + 7) / 8;
    bits = bytes * 8;

    const size_t init_size = dst->size();
    dst->resize(init_size + bytes, 0);
    dst->push_back(static_cast<char>(k_));  // Remember # of probes in filter，在filter最后的字节位压入hash函数个数
    // 对于每个key，使用double-hashing生产一系列的hash值h(K_个)，设置bits array的第h位=1。
    char* array = &(*dst)[init_size];
    for (int i = 0; i < n; i++) {
      // Use double-hashing to generate a sequence of hash values.
      // See analysis in [Kirsch,Mitzenmacher 2006].
      uint32_t h = BloomHash(keys[i]);
      const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
      for (size_t j = 0; j < k_; j++) {
        const uint32_t bitpos = h % bits;
        array[bitpos / 8] |= (1 << (bitpos % 8));
        h += delta;
      }
    }
  }

  bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const override {
    const size_t len = bloom_filter.size();
    if (len < 2) return false;

    const char* array = bloom_filter.data();
    const size_t bits = (len - 1) * 8;

    // Use the encoded k so that we can read filters generated by
    // bloom filters created using different parameters.
    const size_t k = array[len - 1];
    if (k > 30) {
      // Reserved for potentially new encodings for short bloom filters.
      // Consider it a match.
      return true;
    }

    uint32_t h = BloomHash(key);
    const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
    for (size_t j = 0; j < k; j++) {
      const uint32_t bitpos = h % bits;
      if ((array[bitpos / 8] & (1 << (bitpos % 8))) == 0) return false;
      h += delta;
    }
    return true;
  }

 private:
  size_t bits_per_key_; // hashtable的大小，平均意义上，冲突概率是 1/ bits_per_key_
  size_t k_;            // 模拟的hash函数个数
};
```

## "filter" Meta Block

> See doc/table_format.md for an explanation of the filter block format.

If a `FilterPolicy` was specified when the database was opened, a filter block is stored in each table.  The "metaindex" block contains an entry that maps from `filter.<N>` to the BlockHandle for the filter block where `<N>` is the string returned by the filter policy's `Name()` method.

The filter block stores a sequence of filters, where filter i contains the output of `FilterPolicy::CreateFilter()` on all keys that are stored in a block whose file offset falls within the range

    [ i*base ... (i+1)*base-1 ]

Currently, "base" is 2KB.  So for example, if blocks X and Y start in the range `[ 0KB .. 2KB-1 ]`, all of the keys in X and Y will be converted to a filter by calling `FilterPolicy::CreateFilter()`, and the resulting filter will be stored as the first filter in the filter block.

The filter block is formatted as follows:

    [filter 0]
    [filter 1]
    [filter 2]
    ...
    [filter N-1]

    [offset of filter 0]                  : 4 bytes
    [offset of filter 1]                  : 4 bytes
    [offset of filter 2]                  : 4 bytes
    ...
    [offset of filter N-1]                : 4 bytes

    [offset of beginning of offset array] : 4 bytes
    lg(base)                              : 1 byte

The offset array at the end of the filter block allows efficient mapping from a data block offset to the corresponding filter.

## 创建`FilterBlock`

> table/filter_block.h

```cpp
// A FilterBlockBuilder is used to construct all of the filters for a
// particular Table.  It generates a single string which is stored as
// a special block in the Table.
//
// The sequence of calls to FilterBlockBuilder must match the regexp:
//      (StartBlock AddKey*)* Finish
class FilterBlockBuilder {
 public:
  explicit FilterBlockBuilder(const FilterPolicy*);

  FilterBlockBuilder(const FilterBlockBuilder&) = delete;
  FilterBlockBuilder& operator=(const FilterBlockBuilder&) = delete;

  // 开始构建新的filter block，TableBuilder在构造函数和Flush中调用 
  void StartBlock(uint64_t block_offset);

  // 添加key，TableBuilder每次向data block中加入key时调用
  void AddKey(const Slice& key);

  // 结束构建，TableBuilder在结束对table的构建时调用
  Slice Finish();

 private:
  void GenerateFilter();

  const FilterPolicy* policy_;
  std::string keys_;             // Flattened key contents
  std::vector<size_t> start_;    // Starting index in keys_ of each key
  std::string result_;           // Filter data computed so far
  std::vector<Slice> tmp_keys_;  // policy_->CreateFilter() argument
  std::vector<uint32_t> filter_offsets_;
};
```

```cpp
// Generate new filter every 2KB of data
static const size_t kFilterBaseLg = 11;
static const size_t kFilterBase = 1 << kFilterBaseLg;
```

几个函数实现比较简单，这里不再赘述。

## 读取`FilterBlock`

```cpp
class FilterBlockReader {
 public:
  // REQUIRES: "contents" and *policy must stay live while *this is live.
  FilterBlockReader(const FilterPolicy* policy, const Slice& contents);
  bool KeyMayMatch(uint64_t block_offset, const Slice& key);

 private:
  const FilterPolicy* policy_;
  const char* data_;    // Pointer to filter data (at block-start)
  const char* offset_;  // Pointer to beginning of offset array (at block-end)
  size_t num_;          // Number of entries in offset array
  size_t base_lg_;      // Encoding parameter (see kFilterBaseLg in .cc file)
};

// 构造Reader 与 创建时：GenerateFilter+Finish 是整体反序的
FilterBlockReader::FilterBlockReader(const FilterPolicy* policy,
                                     const Slice& contents)
    : policy_(policy), data_(nullptr), offset_(nullptr), num_(0), base_lg_(0) {
  size_t n = contents.size();
  if (n < 5) return;  // 1 byte for base_lg_ and 4 for start of offset array
  base_lg_ = contents[n - 1]; // 最后1byte存的是base
  uint32_t last_word = DecodeFixed32(contents.data() + n - 5);  //偏移数组的位置
  if (last_word > n - 5) return;
  data_ = contents.data();
  offset_ = data_ + last_word;  // 偏移数组开始指针
  num_ = (n - 5 - last_word) / 4; // 计算出filter个数 
}

bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, const Slice& key) {
  uint64_t index = block_offset >> base_lg_;
  if (index < num_) {
    uint32_t start = DecodeFixed32(offset_ + index * 4);
    uint32_t limit = DecodeFixed32(offset_ + index * 4 + 4);
    if (start <= limit && limit <= static_cast<size_t>(offset_ - data_)) {
      Slice filter = Slice(data_ + start, limit - start);
      return policy_->KeyMayMatch(key, filter);
    } else if (start == limit) {
      // Empty filters do not match any keys
      return false;
    }
  }
  return true;  // Errors are treated as potential matches
}
```