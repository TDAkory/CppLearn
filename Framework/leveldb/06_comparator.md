# Comparator

> 理解了Key，再看看Key之间的比较

```cpp
// A Comparator object provides a total order across slices that are
// used as keys in an sstable or a database.  A Comparator implementation
// must be thread-safe since leveldb may invoke its methods concurrently
// from multiple threads.
class LEVELDB_EXPORT Comparator {
 public:
  virtual ~Comparator();

  // Three-way comparison.  Returns value:
  //   < 0 iff "a" < "b",
  //   == 0 iff "a" == "b",
  //   > 0 iff "a" > "b"
  virtual int Compare(const Slice& a, const Slice& b) const = 0;

  // The name of the comparator.  Used to check for comparator
  // mismatches (i.e., a DB created with one comparator is
  // accessed using a different comparator.
  //
  // The client of this package should switch to a new name whenever
  // the comparator implementation changes in a way that will cause
  // the relative ordering of any two keys to change.
  //
  // Names starting with "leveldb." are reserved and should not be used
  // by any clients of this package.
  virtual const char* Name() const = 0;

  // Advanced functions: these are used to reduce the space requirements
  // for internal data structures like index blocks.

  // If *start < limit, changes *start to a short string in [start,limit).
  // Simple comparator implementations may return with *start unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  virtual void FindShortestSeparator(std::string* start,
                                     const Slice& limit) const = 0;

  // Changes *key to a short string >= *key.
  // Simple comparator implementations may return with *key unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  virtual void FindShortSuccessor(std::string* key) const = 0;
};
```

实现类有两个，一个是内置的`BytewiseComparatorImpl`，另一个是`InternalKeyComparator`。

## BytewiseComparatorImpl

```cpp
virtual void FindShortestSeparator(std::string* start, 
                                   onst Slice& limit) const 
{
  // 首先计算共同前缀字符串的长度
  size_t min_length = std::min(start->size(), limit.size());
  size_t diff_index = 0;
  while ((diff_index < min_length) &&
        ((*start)[diff_index] == limit[diff_index]))
  {
     diff_index++;
  }
  if (diff_index >= min_length) 
  {
     // 说明*start是limit的前缀，或者反之，此时不作修改，直接返回
  } 
  else 
  {
     // 尝试执行字符start[diff_index]++，
        设置start长度为diff_index+1，并返回
     // ++条件：字符< oxff 并且字符+1 < limit上该index的字符
     uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);
     if (diff_byte < static_cast<uint8_t>(0xff) &&
         diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) 
     {
         (*start)[diff_index]++;
         start->resize(diff_index + 1);
          assert(Compare(*start, limit) < 0);
     }
  }
}
```

```cpp
void FindShortSuccessor(std::string* key) const override {
    // 找到第一个可以++的字符，执行++后，截断字符串；
    // 如果找不到说明*key的字符都是0xff啊，那就不作修改，直接返回
    size_t n = key->size();
    for (size_t i = 0; i < n; i++) {
      const uint8_t byte = (*key)[i];
      if (byte != static_cast<uint8_t>(0xff)) {
        (*key)[i] = byte + 1;
        key->resize(i + 1);
        return;
      }
    }
    // *key is a run of 0xffs.  Leave it alone.
  }
```

## InternalKeyComparator

`InternalKey`是由`user key`和`sequence number`和`value type`组合而成的。

```cpp
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```

由此可见其排序比较依据依次是：

* 首先根据user key按升序排列
* 然后根据sequence number按降序排列
* 最后根据value type按降序排列

```cpp
void InternalKeyComparator::FindShortestSeparator(std::string* start,
                                                  const Slice& limit) const {
  // Attempt to shorten the user portion of the key
  Slice user_start = ExtractUserKey(*start);
  Slice user_limit = ExtractUserKey(limit);
  std::string tmp(user_start.data(), user_start.size());
  user_comparator_->FindShortestSeparator(&tmp, user_limit);
  if (tmp.size() < user_start.size() &&
      user_comparator_->Compare(user_start, tmp) < 0) {
    // User key has become shorter physically, but larger logically.
    // Tack on the earliest possible number to the shortened user key.
    // user key在物理上长度变短了，但其逻辑值变大了.生产新的*start时，
    // 使用最大的sequence number，以保证排在相同user key记录序列的第一个
    PutFixed64(&tmp,
               PackSequenceAndType(kMaxSequenceNumber, kValueTypeForSeek));
    assert(this->Compare(*start, tmp) < 0);
    assert(this->Compare(tmp, limit) < 0);
    start->swap(tmp);
  }
}
```

```cpp
void InternalKeyComparator::FindShortSuccessor(std::string* key) const {
  Slice user_key = ExtractUserKey(*key);
  std::string tmp(user_key.data(), user_key.size());
  user_comparator_->FindShortSuccessor(&tmp);
  if (tmp.size() < user_key.size() &&
      user_comparator_->Compare(user_key, tmp) < 0) {
    // User key has become shorter physically, but larger logically.
    // Tack on the earliest possible number to the shortened user key.
    PutFixed64(&tmp,
               PackSequenceAndType(kMaxSequenceNumber, kValueTypeForSeek));
    assert(this->Compare(*key, tmp) < 0);
    key->swap(tmp);
  }
}
```